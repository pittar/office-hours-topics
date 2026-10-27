# Encryption on OpenShift

> **Scope:** Red Hat OpenShift Container Platform 4.22 · **Last reviewed:** 2026-10-27
> Architect-level reference. Snippets are illustrative. Check the linked docs before using them in production.

## Summary

- Encryption on OpenShift falls into three areas:
  - **In transit:** north-south ingress traffic and east-west service-to-service traffic.
  - **At rest:** etcd, node disks and persistent volumes.
  - **In use:** confidential computing.
- A lot is encrypted by default, including control plane TLS, cloud-provider volume encryption and RHCOS boot disks on many platforms. But "encrypted by default" rarely means *customer-managed keys* or *end-to-end*. Most customer discussions are about closing those gaps.

---

## 1. Encryption in transit: north-south (ingress)

### Choosing an ingress API

| Option | When to use it | Notes |
|---|---|---|
| **Route** (HAProxy Ingress Controller) | Default for most OpenShift workloads | Mature and OpenShift-native. Edge, re-encrypt and passthrough termination. |
| **Gateway API** | Role-oriented, portable, upstream-standard ingress | Managed by the Ingress Operator, which uses an Istio/OSSM 3-based Envoy gateway. |
| **Kubernetes Ingress** | Portability with existing manifests | OpenShift converts Ingress objects into Routes. |

### Route TLS termination types

| Type | Where TLS ends | Router sees plaintext? | Router → pod | Typical use |
|---|---|---|---|---|
| **edge** | Router (HAProxy) | Yes | **HTTP (unencrypted)** | Simple apps. The internal network is trusted. |
| **reencrypt** | Router, then a new TLS session to the pod | Yes | **HTTPS** | Default recommendation when you need end-to-end encryption *and* L7 features (HSTS, cookies, path routing) |
| **passthrough** | The pod | No (SNI only) | TLS, end to end from the client | The app must own the cert (mTLS to the client, compliance mandates). No L7 features. |

- `insecureEdgeTerminationPolicy`: `Redirect` (recommended), `Allow` or `None`. Applies to edge and re-encrypt only.
- **Service serving certificates:** annotate a Service so that OpenShift issues it a cert signed by the cluster service CA. For re-encrypt routes to such a Service, `destinationCACertificate` can be omitted because the router trusts the service CA.
- **Externally managed certificates:** `spec.tls.externalCertificate` references a `kubernetes.io/tls` Secret in the same namespace as the route. This pairs well with **cert-manager Operator for Red Hat OpenShift**. The router needs RBAC (`get`/`list`/`watch`) on the Secret.
- **HSTS** is set through the `haproxy.router.openshift.io/hsts_header` annotation. It works on edge and re-encrypt routes only. Admins can require it across the cluster with `requiredHSTSPolicies` on the `Ingress` config.

```yaml
# Re-encrypt route that uses a cert-manager-issued external certificate
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend
  annotations:
    haproxy.router.openshift.io/hsts_header: max-age=31536000;includeSubDomains;preload
spec:
  host: app.apps.example.com
  to:
    kind: Service
    name: frontend
  port:
    targetPort: https
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
    externalCertificate:
      name: frontend-tls          # kubernetes.io/tls Secret
    # destinationCACertificate omitted: backend uses a service-serving cert
---
# Backend Service gets a cert signed by the OpenShift service CA
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    service.beta.openshift.io/serving-cert-secret-name: frontend-serving-cert
spec:
  selector: { app: frontend }
  ports:
    - name: https
      port: 8443
      targetPort: 8443
```

### Gateway API

- To enable it, create a `GatewayClass` with `controllerName: openshift.io/gateway-controller/v1`. **This is the only accepted value.** The Ingress Operator then installs a lightweight Istio control plane, based on OSSM 3, in `openshift-ingress`.
- **Gotcha:** if an **OSSM 2.x subscription** is present, the OSSM 3 install fails and Gateway resources have no effect. Check for one before you enable Gateway API.
- TLS terminates on the **Gateway listener**, and certificates are referenced as Secrets through `certificateRefs`. This splits responsibilities cleanly: the platform team owns the Gateway and certificates, and app teams own the `HTTPRoute`s.
- For advanced API management on top of Gateway API (TLS, auth, rate limiting, DNS policies), see **Red Hat Connectivity Link**.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-default
  listeners:
    - name: https
      hostname: "*.gwapi.example.com"
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: gwapi-wildcard-tls   # Secret in openshift-ingress
      allowedRoutes:
        namespaces:
          from: All
```

### TLS security profiles

- Profiles: `Old`, `Intermediate`, `Modern` and `Custom`, based on Mozilla's recommendations.
- **Intermediate (TLS 1.2+) is the default** for the control plane and kubelet. An Ingress Controller with no profile set **inherits the API server's profile**.
- The profile is set in three places: `APIServer` (control plane components, etcd, MCS), `IngressController` and `KubeletConfig`.
- `Modern` (TLS 1.3 only) is marked **"currently unsupported"** for the IngressController in the 4.22 API docs. Validate it before promising it to customers.
- On **FIPS-enabled** clusters (4.22), the Ingress Operator filters out non-compliant TLS 1.3 ciphers. Remember that FIPS mode is an **install-time** choice.

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  tlsSecurityProfile:
    type: Intermediate
    intermediate: {}
```

---

## 2. Encryption in transit: east-west (OpenShift Service Mesh 3)

- OSSM 3 is based on upstream Istio and managed by the **Sail-based operator**. The `Istio` CR replaces the OSSM 2 `ServiceMeshControlPlane`.
- **Auto mTLS is on by default.** Sidecar-to-sidecar traffic is encrypted even in `PERMISSIVE` mode.
- **`PERMISSIVE` is the default `PeerAuthentication` mode**, which means plaintext traffic is *also* accepted. Use `STRICT` to *require* mTLS.
  - Namespace scope: a `PeerAuthentication` named `default` in the app namespace.
  - Mesh scope: the same object in the istiod namespace, which must match `spec.namespace` of the `Istio` CR.
- **Migrating from OSSM 2:** `spec.security.dataPlane.mtls` no longer exists. Replace it with `PeerAuthentication` and `DestinationRule` objects.
- **Workload certificates** come from istiod's CA by default. To use a corporate PKI, integrate with **cert-manager** (istio-csr).
- Roll out in stages: use `PERMISSIVE`, confirm traffic in Kiali that mTLS is in use, switch to `STRICT`, then layer `AuthorizationPolicy` on top for identity-based access.
- Without a mesh, pod-to-pod traffic is not encrypted by the application layer. For node-to-node encryption, look at **OVN-Kubernetes IPsec**, which encrypts traffic between nodes rather than per workload.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system   # mesh-wide; use an app namespace for namespace scope
spec:
  mtls:
    mode: STRICT
```

---

## 3. Encryption at rest

### What is encrypted by default?

| Layer | Default on OCP 4.22 | Customer action |
|---|---|---|
| **etcd** (Secrets, ConfigMaps, Routes, OAuth tokens, …) | **Not encrypted** at the application layer | Set `APIServer.spec.encryption.type` to `aesgcm` or `aescbc`. KMS v2 is Tech Preview. |
| **RHCOS boot disks** | Depends on platform and installer config | Use TPM v2 and/or Tang (NBDE) through Butane/MachineConfig at **install time** |
| **PVCs on AWS (EBS CSI)** | **Encrypted.** The default storage classes set `encrypted: "true"` and use the AWS-managed `aws/ebs` key. | Use a customer key: `kmsKeyId` in a StorageClass or `kmsKeyARN` on the ClusterCSIDriver |
| **PVCs on Azure / GCP** | Encrypted by the cloud provider with platform-managed keys | Customer-managed keys through the cloud KMS (Azure Disk Encryption Set, Google Cloud KMS) |
| **PVCs on vSphere / bare metal / LSO** | **Not encrypted** by OpenShift | vSphere VM encryption policies, ODF encryption or encrypted backing storage |
| **ODF (Ceph) PVCs** | Not encrypted unless enabled | Cluster-wide encryption (block and file) and/or per-StorageClass encryption (block only) using an external KMS |

### Key points

- **Cloud default encryption protects against stolen physical media. It does not protect against a cloud or cluster admin.** If compliance requires customer key control, use customer-managed keys.
- **etcd encryption:**
  - Keys rotate every 7 days, and up to 10 previous keys are kept so older backups can still be decrypted.
  - Switching the `type` migrates existing data automatically.
  - **KMS v2** (`type: KMS`) is **Tech Preview** in 4.22. It requires `TechPreviewNoUpgrade` and currently a HashiCorp Vault Enterprise plugin. Don't propose it for production.
  - Back up etcd, and protect the backup, which contains the encryption keys.
- **ODF:**
  - Supported KMS backends are HashiCorp Vault and Thales CipherTrust Manager.
  - KMS integration requires an ODF Advanced subscription.
  - Key rotation can be enabled on the `StorageCluster`.
- **Boot disk (LUKS2):**
  - TPM v2 is the preferred mode. Tang is supported only for bare metal and vSphere UPI.
  - Set `threshold` to require both (threshold = number of Tang servers + 1).
  - It can't easily be added after install. Plan for it before the cluster is built.
- **Remember the Secrets topic.** Encrypting etcd protects the data on disk, but anyone with RBAC access can still read Secrets through the API (see [Secrets Management](secretsmanagement.md)).

```yaml
# etcd encryption (control plane rolls out; allow time for migration)
apiVersion: config.openshift.io/v1
kind: APIServer
metadata:
  name: cluster
spec:
  encryption:
    type: aesgcm
---
# AWS: StorageClass using a customer-managed KMS key
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-csi-cmk
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:111122223333:key/<key-id>
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# Butane: boot disk encryption with TPM v2 + Tang (render with `butane` to a MachineConfig)
variant: openshift
version: 4.22.0
metadata:
  name: worker-storage
  labels:
    machineconfiguration.openshift.io/role: worker
boot_device:
  layout: x86_64
  luks:
    tpm2: true
    tang:
      - url: http://tang.example.com:7500
        thumbprint: <thumbprint>
    threshold: 1
```

---

## 4. Encryption in use: confidential computing (overview)

- **The problem:** data in memory is visible to the host, hypervisor and infrastructure admins. **Trusted Execution Environments (TEEs)**, such as AMD SEV-SNP and Intel TDX, encrypt memory in hardware.
- **Confidential Containers (CoCo)** are delivered through the **OpenShift sandboxed containers** operator:
  - `kata-cc` runtime class: bare metal with TEE-capable CPUs.
  - `kata-remote` runtime class: peer pods running as confidential VMs on Azure or other supported clouds.
- **Red Hat build of Trustee** handles **attestation**. It verifies the TEE before releasing keys or secrets to the workload. Red Hat recommends running it on a **separate, trusted cluster**.
- **Good fits:** regulated or multi-party data, AI models and data in untrusted or public infrastructure, sovereign cloud.
- **Caveats:** hardware and cloud instance requirements, a performance cost, and support levels that vary by platform and topology (for example, hosted control planes are Tech Preview).

---

## Common questions and gotchas

- **"Is my data encrypted in OpenShift?"** Split the question by layer: ingress, east-west, etcd, node disks, PVs and memory. Each layer has a different default.
- **Edge vs. re-encrypt:** with edge routes, traffic from the router to the pod is plaintext. If a regulation says "encrypted end to end", use re-encrypt or passthrough.
- **Passthrough loses L7 features:** no HSTS annotation, cookies, path-based routing or header manipulation at the router.
- **Gateway API vs. OSSM 2.x:** they conflict. Plan the OSSM 3 migration first.
- **Certificate lifecycle** causes more outages than encryption itself. Standardize on cert-manager, and monitor certificate expiry.
- **FIPS** is set at install time and needs RHEL/RHCOS FIPS-validated modules, with `fips: true` in `install-config.yaml`. It can't be turned on later.

## Discovery questions

- Which compliance frameworks apply (PCI-DSS, HIPAA, FedRAMP, DORA, …)? Do they require **customer-managed keys** or **FIPS**?
- Is "encrypted in transit" interpreted as client → router, or as client → pod?
- Which platforms are involved (cloud, vSphere, bare metal)? Is ODF in use?
- Is there an existing enterprise PKI or KMS (Vault, CipherTrust, cloud KMS) to integrate with?
- Are workloads running in untrusted, shared or sovereign infrastructure where confidential computing applies?

## References (OpenShift 4.22)

- [Routes (secured routes, external certificates)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ingress_and_load_balancing/routes)
- [Configuring Gateway API](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ingress_and_load_balancing/configuring-gateway-api)
- [TLS security profiles](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/security_and_compliance/tls-security-profiles)
- [etcd encryption](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html-single/etcd/index)
- [Persistent storage (AWS EBS encryption)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/storage/configuring-persistent-storage)
- [Installation configuration (boot disk encryption)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/pdf/installation_configuration/index)
- [OSSM 3: Enabling mTLS](https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.2/html/installing/ossm-enabling-mtls)
- [OpenShift sandboxed containers (Confidential Containers, Trustee)](https://docs.redhat.com/en/documentation/openshift_sandboxed_containers/1.13)
- [OCP 4.22 release notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/release_notes/ocp-4-22-release-notes)
