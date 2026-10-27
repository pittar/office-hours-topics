# Secrets Management on OpenShift

> **Scope:** Red Hat OpenShift Container Platform 4.22 · **Last reviewed:** 2026-10-27
> Architect-level reference. Snippets are illustrative. Check the linked docs before using them in production.

## Summary

- Native Kubernetes `Secret` objects are **base64-encoded, not encrypted**. Anyone with RBAC `get` access can read them, and they sit in etcd as plaintext unless etcd encryption is enabled (see [Encryption](encryption.md)).
- Most enterprises want the **source of truth outside the cluster**, in Vault, a cloud secrets manager or similar. That gives central audit, rotation and lifecycle management.
- OpenShift 4.22 provides two Red Hat-supported operators to bridge the external store and workloads:
  - **External Secrets Operator for Red Hat OpenShift (ESO):** syncs external secrets into native Kubernetes Secrets.
  - **Secrets Store CSI Driver Operator (SSCSI):** mounts external secrets directly into pods as files.

---

## 1. Choosing between ESO and the Secrets Store CSI driver

| | **External Secrets Operator** | **Secrets Store CSI Driver** |
|---|---|---|
| How it delivers secrets | Reconciles a **Kubernetes Secret** from the external store | Mounts secrets as **files in a pod volume** (tmpfs) when the pod starts |
| Stored in etcd? | **Yes**, as a normal Secret | **No**, unless you opt in to `secretObjects` sync |
| App changes needed | None. Works with `envFrom`, `secretKeyRef` and any operator that expects a Secret | The app reads files, or you enable sync to get env vars |
| Secret exists when | Independently of pods, as soon as the ExternalSecret reconciles | Only while a pod that mounts it is running. Synced Secrets are deleted when the last such pod goes away. |
| Non-pod consumers (Routes, image pull secrets, operators, GitOps) | **Yes** | Not directly |
| Refresh | `refreshInterval` per ExternalSecret | Remounting or rotation of the mounted content. **The app must re-read the files.** |
| Write back to the store | `PushSecret` (check its support status) | No |
| Tested providers (4.22 docs) | AWS Secrets Manager, HashiCorp Vault, Google Secret Manager, Azure Key Vault, IBM Cloud Secrets Manager (plus a Bitwarden plugin) | AWS Secrets Manager, AWS SSM Parameter Store, Azure Key Vault, Google Secret Manager, HashiCorp Vault |

**Rules of thumb**
- **Use ESO by default.** It is the most flexible option, needs no app changes, works well with GitOps, and covers platform consumers such as pull secrets and TLS secrets.
- **Use SSCSI** when policy says **"secrets must not be stored in etcd or the Kubernetes API"**, and the app can read files.
- **Both can run on the same cluster.** Many customers use ESO for platform and app config, and SSCSI for their highest-sensitivity workloads.

---

## 2. External Secrets Operator for Red Hat OpenShift

- Red Hat's supported build of the upstream external-secrets project.
  - **1.0 was the first GA release**. The latest at the time of writing is **1.2.1**, based on upstream v2.5.0.
  - Install it from OperatorHub (channel `stable-v1`).
- **Namespaces:** the operator runs in `external-secrets-operator`. The operand (core controller, webhook and cert-controller) runs in `external-secrets`.
- **The operand is configured through a single `ExternalSecretsConfig` CR named `cluster`.**
- **Key custom resources:**
  - **`SecretStore` / `ClusterSecretStore`:** how to connect and authenticate to the provider. A SecretStore is namespace-scoped; a ClusterSecretStore is cluster-wide.
  - **`ExternalSecret`:** which keys to fetch and the target Secret to create.
  - `ClusterExternalSecret` and `PushSecret` are available upstream. Check their Red Hat support status before you recommend them.
- **Network policy is on by default.** The operand denies egress except to required services. **You must add an egress allow rule for your provider**, or the stores won't reach it.
- **cert-manager:** the webhook can use cert-manager. **Install and verify the cert-manager Operator first**, or restart the ESO operator pod afterward.
- **FIPS:** supported when ESO is installed on a FIPS-mode cluster.
- **Only one ESO per cluster.** Uninstall the community ESO before installing the Red Hat build; the docs describe the migration.
- **Uninstalling** the operator does **not** remove the `external-secrets` resources. You have to clean them up manually.

```yaml
# Operand config: allow controller egress to HTTPS (narrow it to your provider where possible)
apiVersion: operator.openshift.io/v1alpha1
kind: ExternalSecretsConfig
metadata:
  name: cluster
spec:
  controllerConfig:
    networkPolicies:
      - name: allow-external-secrets-egress
        componentName: ExternalSecretsCoreController
        egress:
          - ports:
              - port: 443
                protocol: TCP
---
# Cluster-wide store backed by Vault, using Kubernetes auth (no static token)
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: https://vault.example.com:8200
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: eso
          serviceAccountRef:
            name: eso-vault-auth
            namespace: external-secrets
---
# App team requests a secret; ESO creates and refreshes Secret "db-creds"
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-creds
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: vault
  target:
    name: db-creds
  data:
    - secretKey: password
      remoteRef:
        key: my-app/db
        property: password
```

---

## 3. Secrets Store CSI Driver Operator

- Part of OpenShift, installed from OperatorHub.
  - In 4.22 it is based on upstream secrets-store-csi-driver **v1.5.6**, and IBM Z support was added (Vault provider only on IBM Z).
- **The operator ships the driver only.** You install the **provider plugin**, such as the Vault CSI provider or the AWS provider, separately. Those plugins fall under Red Hat's **third-party support policy**.
- **Disconnected clusters:** supported for AWS Secrets Manager, Azure Key Vault, Google Secret Manager and Vault, provided the cluster has private connectivity to the provider and to OIDC/STS.
- **How it works:**
  1. Create a `SecretProviderClass` per namespace that names the provider and the objects to fetch.
  2. Pods reference it in a CSI inline volume.
  3. Content is fetched **when the pod starts** and mounted read-only.
- A `SecretProviderClassPodStatus` object in the namespace shows which objects and versions are mounted in each pod.
- **Optional sync to a Kubernetes Secret** (`secretObjects`):
  - Use it for env vars or for apps that expect a Secret.
  - It **puts the secret back into etcd**, so enable it only when you need it.
- **Workload identity is the preferred authentication method:**
  - AWS STS (IRSA-style), Azure Workload Identity, GCP Workload Identity.
  - Vault Kubernetes auth bound to the pod's ServiceAccount.

```yaml
# Enable the driver after installing the operator
apiVersion: operator.openshift.io/v1
kind: ClusterCSIDriver
metadata:
  name: secrets-store.csi.k8s.io
spec:
  managementState: Managed
---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db
  namespace: my-app
spec:
  provider: vault
  parameters:
    vaultAddress: https://vault.example.com:8200
    roleName: my-app
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/my-app/db"
        secretKey: "password"
  # Optional: also sync to a Kubernetes Secret (stored in etcd)
  # secretObjects:
  #   - secretName: db-creds
  #     type: Opaque
  #     data:
  #       - objectName: db-password
  #         key: password
---
# Pod spec excerpt: secret appears at /mnt/secrets/db-password
spec:
  serviceAccountName: my-app
  containers:
    - name: app
      volumeMounts:
        - name: secrets
          mountPath: /mnt/secrets
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: vault-db
```

---

## 4. Patterns and recommendations

- **GitOps:** commit `ExternalSecret` and `SecretProviderClass` manifests to Git, never secret values. Both work cleanly with **Red Hat OpenShift GitOps** (Argo CD).
  - Sealed Secrets and SOPS are community alternatives. They put encrypted secrets *in* Git, which comes with its own key management burden.
- **The "secret zero" problem:**
  - **Avoid static tokens in SecretStores.** Prefer short-lived credentials: Vault Kubernetes or JWT auth, cloud workload identity (STS, Azure, GCP), or IBM Cloud IAM.
  - Remember that **ESO output is still a Kubernetes Secret.** Pair it with etcd encryption and tight RBAC.
- **RBAC:**
  - Platform team: owns `ClusterSecretStore` objects and provider auth.
  - App teams: create `ExternalSecret`s only. Limit who can create or modify `SecretStore`s, because a store is effectively a credential to the vault.
- **Rotation:** rotating at the source isn't enough. Apps must reload, either through file watches (SSCSI) or by being restarted when the Secret changes (ESO).
- **Certificates** are a separate concern: use the **cert-manager Operator for Red Hat OpenShift** (see [Encryption](encryption.md)).

## Common questions and gotchas

- **"Are Kubernetes Secrets encrypted?"** Not by default, and base64 is not encryption. Enable etcd encryption (`aesgcm`/`aescbc`) and restrict RBAC either way.
- **ESO can't reach the provider after install:** this is almost always the default-deny NetworkPolicy. Add an egress rule in `ExternalSecretsConfig`.
- **SSCSI pod stuck in `ContainerCreating`:** usually the provider plugin isn't installed, auth or a role binding is wrong, or the secret path is wrong. Check the pod events and the provider pod logs.
- **SSCSI synced Secret disappeared:** expected behavior, because it is deleted when no pods mount it. Use ESO if the Secret must persist.
- **Community and Red Hat ESO both installed:** not supported. Migrate to the Red Hat build.

## Discovery questions

- What is the system of record today: Vault, cloud secrets manager, CyberArk, spreadsheets? Is there more than one?
- Is there a policy that **secrets must not be stored in etcd** or the Kubernetes API?
- How do workloads authenticate to the store today? Can you use workload identity instead of static tokens?
- What are the rotation requirements, and can the applications reload secrets without restarting?
- Is the environment disconnected? Is GitOps the deployment model?

## References (OpenShift 4.22)

- [External Secrets Operator for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/security_and_compliance/external-secrets-operator-for-red-hat-openshift)
- [Understanding secrets management in OpenShift (4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/security_and_compliance/understanding-secrets-management)
- [Secrets Store CSI Driver Operator (Storage, CSI)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/storage/using-container-storage-interface-csi)
- [Providing sensitive data to pods by using an external secrets store (Nodes, 4.20)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/nodes/working-with-pods)
- [OpenShift GitOps: Managing secrets with the Secrets Store CSI driver](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.16/html/security/managing-secrets-securely-using-sscsid-with-gitops)
- [OCP 4.22 release notes](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/release_notes/ocp-4-22-release-notes)
