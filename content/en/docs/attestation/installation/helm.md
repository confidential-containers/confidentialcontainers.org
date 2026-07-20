---
title: Trustee with Helm
description: Installing Trustee with Helm
weight: 15
categories:
- attestation
tags:
- trustee
- attestation
- installation
- kubernetes
---

Use the Helm chart to deploy Trustee on Kubernetes when you need production-grade,
declarative control over the deployment. For example, persistent storage, your own keys,
or IBM Secure Execution support.

The chart deploys three components (KBS, gRPC Attestation Service (AS), and RVPS) and can
optionally bundle PostgreSQL (via the [Bitnami chart](https://artifacthub.io/packages/helm/bitnami/postgresql)).
KBS is wired to the remote `coco_as_grpc` Attestation Service.


{{% alert title="Note" color="info" %}}
The Helm commands on this page deploy the latest Helm chart.
{{% /alert %}}

## Prerequisites

- Kubernetes 1.19+
- Helm 3
- For PostgreSQL storage (`storageBackend.type: Postgres` or `sessionStorageType: Postgres`): the Bitnami subchart uses PVC-backed storage, so your cluster must provide a usable `StorageClass`, or you must bind an existing claim.

## Install

1. Clone the repository:

    ```bash
    git clone https://github.com/confidential-containers/trustee.git
    ```

1. Navigate into the repository:

    ```bash
    cd trustee
    ```

1. Update the Helm chart:

    ```bash
    helm dependency update ./deployment/helm-chart
    ```

1. Deploy the Helm chart:

    ```bash
    helm upgrade --install trustee ./deployment/helm-chart \
     --namespace coco-trustee --create-namespace
    ```

    **Expected output:**

    ```bash
    Release "trustee" does not exist. Installing it now.
    NAME: trustee
    LAST DEPLOYED: Sat Jul 18 02:47:16 2026
    NAMESPACE: coco-trustee
    STATUS: deployed
    REVISION: 1
    DESCRIPTION: Install complete
    TEST SUITE: None
    ```

1. Check the status of the deployment:

    ```bash
    kubectl get pods -n coco-trustee -w
    ```

    **Expected output:**

    ```bash
    NAME                           READY   STATUS     RESTARTS   AGE
    trustee-as-74b4c898d8-rkdvk    1/1     Running    0          19s
    trustee-kbs-d7cb6d957-7ct47    1/1     Running    0          19s
    trustee-rvps-7d6c58d8f-mknqd   1/1     Running    0          19s
    ```

    Wait for workloads to be in a running state before moving on to the next step.

1. Port-forward KBS (default HTTP **8080**). The internal ClusterIP Service is `<Helm fullname>-kbs` (with the install command above, `trustee-kbs`):

    ```bash
    kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080
    ```

## Helm configuration options

The default `values.yaml` is intentionally small. Fixed on-disk paths for LocalFs or LocalJson are defined in `templates/_helpers.tpl` and are not overridable via values. You can merge extra keys with `-f` / `--set` (Helm merges arbitrary values).

Refer to the [Trustee repository](https://github.com/confidential-containers/trustee/tree/main/deployment/helm-chart#values) for full details on available Helm configuration options.

## Typical installation scenarios

The following are some typical installation scenarios for Trustee with Helm.

{{% alert title="Note" color="info" %}}
Example Helm `values.yaml` files are available in the [/scenarios folder](https://github.com/confidential-containers/trustee/tree/main/deployment/helm-chart/scenarios) in the Trustee repository.
{{% /alert %}}

### LocalFs storage (default install)

This is the default install option.
If you do not set `storageBackend.type` or `sessionStorageType` to `Postgres`, the chart does not deploy bundled Postgres. Instead, components use the default `storageBackend` (for example, `LocalFs`) on your cluster.

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
    --namespace coco-trustee --create-namespace
```


### PostgreSQL as storage backend + in-memory KBS sessions

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml
```

This enables the Bitnami PostgreSQL subchart (`postgresql.enabled: true`) and sets `storageBackend.type: Postgres`. KBS sessions stay in memory (`sessionStorageType: Memory`). Demo credentials default to `trustee` / `trustee` / `trustee` (override via `postgresql.auth.*`).

### External PostgreSQL

When an external Postgres service is used, set `storageBackend.postgres.mode=external`, pre-create a Secret with a `POSTGRES_URL` key, and point the chart at it:

```bash
kubectl create secret generic trustee-external-postgres -n coco-trustee \
  --from-literal=POSTGRES_URL='postgresql://user:password@postgres.example.com:5432/trustee?sslmode=require'

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set storageBackend.type=Postgres \
  --set storageBackend.postgres.mode=external \
  --set storageBackend.postgres.external.existingSecretName=trustee-external-postgres \
  --set storageBackend.postgres.external.existingSecretKey=POSTGRES_URL
```

When `storageBackend.postgres.mode=external`, the chart does **NOT** deploy the Bitnami subchart (`postgresql.enabled` stays `false`), even if Postgres is required by `storageBackend.type` or `sessionStorageType`.

### Bring your own keys (BYOK)

Key material is controlled only by `secrets.useEphemeralGeneratedKeys`:

- `true` (default): a Helm pre-install / pre-upgrade hook Job generates ephemeral demo keys into a release-scoped Secret (name ends with `bootstrap-user-keys`). `helm uninstall` runs a post-delete hook that removes that Secret.
- `false`: you must pre-create a Kubernetes `Secret` in the target namespace, then set `secrets.existingSecretName` to that name. The bootstrap hook is not rendered.

When ephemeral generation is enabled, the hook uses:

- an `initContainer` (OpenSSL image) to generate keys into an `emptyDir`
- a `quay.io/kata-containers/kubectl` container to create the Secret from generated files

Both images are overridable via `bootstrapUserKeysJob.keygenImage.*` and `bootstrapUserKeysJob.kubectlImage.*`.

When ephemeral generation is disabled, the Secret must define these data keys (values are PEM text or base64-encoded PEM, same as any `kubectl create secret generic --from-file=...`):

| Secret key | Role |
|------------|------|
| `KBS_ADMIN_PRIVATE_KEY` / `KBS_ADMIN_PUBKEY` | KBS admin API Ed25519 keypair (used to sign admin JWTs). |
| `KBS_ADMIN_TOKEN` | Pre-signed admin bearer JWT for `kbs-client --admin-token-file` (generated by the bootstrap hook when ephemeral keys are enabled). |
| `AS_TOKEN_SIGNING_PRIVATE_KEY` | Attestation Service: sign attestation tokens. |
| `AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN` | AS: `x5c` / cert chain; KBS: trust anchor for token verification. |

The chart mounts that Secret on KBS and gRPC AS and maps those keys to in-container paths `private.key`, `public.pub`, `token.key`, `token-cert-chain.pem` under `/opt/confidential-containers/kbs/user-keys`.

Example (create Secret, then install):

```bash
kubectl create secret generic trustee-byok-keys -n coco-trustee \
  --from-file=KBS_ADMIN_PRIVATE_KEY=./admin.key.pem \
  --from-file=KBS_ADMIN_PUBKEY=./admin.pub.pem \
  --from-file=AS_TOKEN_SIGNING_PRIVATE_KEY=./token.key.pem \
  --from-file=AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN=./token-chain.pem

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  --set secrets.useEphemeralGeneratedKeys=false \
  --set secrets.existingSecretName=trustee-byok-keys
```

Or use `scenarios/bring-your-own-keys.yaml` (adjust Secret name / file paths in the comments there).

### IBM Secure Execution (s390x)

On s390x, the IBM Secure Execution (SE) verifier needs attestation materials at runtime. Because KBS talks to a remote `coco_as_grpc` AS, the verifier runs inside the **AS Pod**, so these materials must be mounted on **AS**, not KBS. (This differs from the builtin-AS kustomize overlay in `kbs/config/kubernetes/overlays/ibm-se`, which mounts them on KBS.)

The verifier reads materials from fixed paths under `/run/confidential-containers/ibmse/` (overridable via `SE_*` env vars; see `deps/verifier/src/se/README.md`). The chart mounts them from a local node path via a PersistentVolume / PersistentVolumeClaim — set `as.verifier.se.credsDir` to the directory on the node that contains the materials (equivalent to `IBM_SE_CREDS_DIR` used in the kustomize overlay), and `as.verifier.se.nodeName` to the name of that node.  The chart then creates a `local`-type PV + PVC and mounts the whole directory at `/run/confidential-containers/ibmse/` on the AS Pod.

| Material | Expected path under `credsDir` | Notes |
|----------|-------------------------------|-------|
| RSA measurement key pair | `rsa/encrypt_key.{pem,pub}` | Private key is sensitive, restrict node access. |
| Signing / intermediate certs | `certs/` | Directory; all files are read. |
| CRLs | `crls/` | Directory; all files are read. |
| Host Key Documents (HKD) | `hkds/` | Directory; all files are read. |
| SE image header | `hdr/hdr.bin` | Binary file. |
| Root CA (optional) | `root_ca.crt` | Single file. |

Set `CERTS_OFFLINE_VERIFICATION=true` (via `as.extraEnvVars`) to verify the HKD certificate chain offline. Do not set `SE_SKIP_CERTS_VERIFICATION=true` outside development, it disables HKD certificate chain verification.

```bash
# 1. Place all materials under a directory on the target s390x node, e.g.:
#    $IBM_SE_CREDS_DIR/{rsa/,certs/,crls/,hkds/,hdr/hdr.bin}
#    See deps/verifier/src/se/README.md for how to obtain the materials.

# 2. Install, pointing the chart at the node and directory:
helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/ibm-se.yaml \
  --set as.verifier.se.credsDir=$IBM_SE_CREDS_DIR \
  --set as.verifier.se.nodeName=<your-s390x-node-name>
```

Use an s390x AS image built with the `se-verifier` feature (`as.image.repository` / `as.image.tag`). See `scenarios/ibm-se.yaml` for the full override. Set the SE attestation policy afterwards as documented in `deps/verifier/src/se/README.md`.

## Validate the deployment

**Inspect resources**:

```bash
kubectl get deploy,pods,svc -n coco-trustee
helm status trustee -n coco-trustee
```

**Render-only check** (no install):

```bash
helm dependency update ./deployment/helm-chart

helm template trustee ./deployment/helm-chart \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml \
  --namespace coco-trustee > /tmp/trustee-render.yaml
```

{{% alert title="Note" color="info" %}}
If your cluster cannot resolve `*.svc.cluster.local` from Pods, set `dnsHostAliasWorkaround: true` in your override values, then run `helm upgrade` again after Services exist so Helm `lookup` can resolve ClusterIPs.
{{% /alert %}}

**Exercise KBS with `kbs-client`**:

Build the client from the repo with `cargo build -p kbs-client --release`. With ephemeral keys, the hook-created Secret (name ends with `bootstrap-user-keys`) includes a pre-signed admin JWT under `KBS_ADMIN_TOKEN`. KBS expects `authorization_mode = "AuthenticatedAuthorization"` with a bearer JWT that includes a `role` claim matching `[admin.authorization.regex_acl]` (default role `admin`).

```bash
kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080 &
SECRET=$(kubectl get secrets -n coco-trustee -o name | grep bootstrap-user-keys | head -1 | cut -d/ -f2)
kubectl get secret "$SECRET" -n coco-trustee -o jsonpath='{.data.KBS_ADMIN_TOKEN}' | base64 -d >/tmp/admin-token
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource-policy --allow-all
```

Set a confidential resource(`config` + `--admin-token-file`, then `set-resource`):

```bash
echo 'demo-payload' >/tmp/demo-resource.txt
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource \
  --path my_repo/resource_type/demo --resource-file /tmp/demo-resource.txt
```

Fetch a resource by KBS URI path (`get-resource` is a top-level subcommand; it follows the normal attestation / token flow for your client build and policy):

```bash
kbs-client --url http://127.0.0.1:8080 get-resource --path my_repo/resource_type/demo
```

## Uninstall

Remove the release:

```bash
helm uninstall trustee -n coco-trustee
```

{{% alert title="Note" color="info" %}}
When `secrets.useEphemeralGeneratedKeys` is `true` (default), a post-delete Helm hook removes the release-scoped `*-bootstrap-user-keys` Secret automatically.
{{% /alert %}}