---
title: Trustee with Helm
description: Installing Trustee on Kubernetes with Helm
weight: 15
categories:
- attestation
tags:
- trustee
- attestation
- installation
- kubernetes
---

Use the Helm chart to deploy Trustee on Kubernetes with declarative control over the
deployment. The chart deploys KBS, the gRPC Attestation Service (AS), and RVPS. It can
also deploy PostgreSQL through the
[Bitnami chart](https://artifacthub.io/packages/helm/bitnami/postgresql). KBS connects
to the remote `coco_as_grpc` Attestation Service.


{{% alert title="Note" color="info" %}}
The Helm commands on this page deploy the latest Helm chart.
{{% /alert %}}

## Prerequisites

- Kubernetes 1.19 or later
- Helm 3
- A usable `StorageClass` or an existing bound claim for PostgreSQL storage (`storageBackend.type: Postgres` or
  `sessionStorageType: Postgres`).
  The Bitnami subchart uses persistent volume claim (PVC)-backed storage.

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

    ```
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

    ```
    NAME                           READY   STATUS     RESTARTS   AGE
    trustee-as-74b4c898d8-rkdvk    1/1     Running    0          19s
    trustee-kbs-d7cb6d957-7ct47    1/1     Running    0          19s
    trustee-rvps-7d6c58d8f-mknqd   1/1     Running    0          19s
    ```

    Wait for all workloads to reach the `Running` state before continuing.

1. Forward local port `8080` to the KBS service. The internal ClusterIP service is
   named `<Helm fullname>-kbs`. For this installation, the name is `trustee-kbs`.

    ```bash
    kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080
    ```

Refer to the [validate the deployment](helm.md#validate-the-deployment) section for details on checking your Trustee instance after instalation.

## Helm configuration options

The default `values.yaml` contains only the required settings for deploying Trustee. For all available
options, see the
[Helm chart values documentation](https://github.com/confidential-containers/trustee/tree/main/deployment/helm-chart#values).

## Deployment scenarios

The following sections show common Trustee deployment scenarios.

{{% alert title="Note" color="info" %}}
Example Helm `values.yaml` files are available in the
[`scenarios` directory](https://github.com/confidential-containers/trustee/tree/main/deployment/helm-chart/scenarios)
in the Trustee repository.
{{% /alert %}}

### LocalFs storage (default)

If neither `storageBackend.type` nor `sessionStorageType` is set to `Postgres`, the
chart does not deploy the bundled PostgreSQL database. Components instead use the
default `storageBackend`, such as `LocalFs`, on your cluster.

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace
```

### PostgreSQL storage with in-memory KBS sessions

```bash
helm dependency update ./deployment/helm-chart

helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml
```

This configuration enables the Bitnami PostgreSQL subchart
(`postgresql.enabled: true`) and sets `storageBackend.type: Postgres`. KBS sessions
remain in memory (`sessionStorageType: Memory`). The default demo database name,
username, and password are all `trustee`. Override them with `postgresql.auth.*`.

### External PostgreSQL

To use an external PostgreSQL service, set `storageBackend.postgres.mode=external`,
create a Secret containing a `POSTGRES_URL` key, and configure the chart to use it:

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

When `storageBackend.postgres.mode=external`, the chart does not deploy the Bitnami
subchart (`postgresql.enabled` remains `false`), even when `storageBackend.type` or
`sessionStorageType` requires PostgreSQL.

### Bring your own keys (BYOK)

The `secrets.useEphemeralGeneratedKeys` setting controls key material:

- `true` (default): A Helm pre-install and pre-upgrade hook Job generates ephemeral
  demo keys in a release-scoped Secret whose name ends with `bootstrap-user-keys`.
  A post-delete hook removes the Secret when you run `helm uninstall`.
- `false`: Create a Kubernetes Secret in the target namespace and set
  `secrets.existingSecretName` to its name. The chart does not render the bootstrap
  hook.

When ephemeral generation is enabled, the hook uses:

- An `initContainer` that uses an OpenSSL image to generate keys in an `emptyDir`.
- A `quay.io/kata-containers/kubectl` container that creates the Secret from the
  generated files.

You can override both images with `bootstrapUserKeysJob.keygenImage.*` and
`bootstrapUserKeysJob.kubectlImage.*`.

When ephemeral key generation is disabled, the Secret must define the following data
keys. As with any `kubectl create secret generic --from-file=...` command, values are
PEM text or base64-encoded PEM:

| Secret key | Role |
|------------|------|
| `KBS_ADMIN_PRIVATE_KEY` / `KBS_ADMIN_PUBKEY` | KBS admin API Ed25519 key pair used to sign admin JWTs |
| `KBS_ADMIN_TOKEN` | Pre-signed admin bearer JWT for `kbs-client --admin-token-file`; the bootstrap hook generates it when ephemeral keys are enabled |
| `AS_TOKEN_SIGNING_PRIVATE_KEY` | Private key that the Attestation Service uses to sign attestation tokens |
| `AS_TOKEN_VERIFICATION_PUBLIC_KEY_CERT_CHAIN` | AS `x5c` certificate chain and KBS trust anchor for token verification |

The chart mounts the Secret in KBS and gRPC AS. It maps the keys to the in-container
paths `private.key`, `public.pub`, `token.key`, and `token-cert-chain.pem` under
`/opt/confidential-containers/kbs/user-keys`.

Create the Secret, and then install Trustee:

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

Alternatively, use `scenarios/bring-your-own-keys.yaml` and adjust the Secret name and
file paths described in its comments.

### IBM Secure Execution (s390x)

On s390x, the IBM Secure Execution (SE) verifier requires attestation materials at runtime. Because KBS connects to a remote `coco_as_grpc` AS, the verifier runs in the AS Pod. Mount these materials on AS, not KBS. This differs from the built-in AS
Kustomize overlay in `kbs/config/kubernetes/overlays/ibm-se`, which mounts them on KBS.

The verifier reads materials from fixed paths under
`/run/confidential-containers/ibmse/`. You can override these paths with `SE_*`
environment variables; refer to `deps/verifier/src/se/README.md`. The chart mounts the materials from a local node path through a persistent volume (PV) and persistent
volume claim (PVC):

- Set `as.verifier.se.credsDir` to the directory on the node that contains the
  materials. This is equivalent to `IBM_SE_CREDS_DIR` in the Kustomize overlay.
- Set `as.verifier.se.nodeName` to the name of that node.

The chart creates a `local` PV and PVC, and then mounts the directory at
`/run/confidential-containers/ibmse/` on the AS Pod.

| Material | Expected path under `credsDir` | Notes |
|----------|-------------------------------|-------|
| RSA measurement key pair | `rsa/encrypt_key.{pem,pub}` | The private key is sensitive; restrict node access |
| Signing and intermediate certificates | `certs/` | Directory; all files are read |
| Certificate revocation lists (CRLs) | `crls/` | Directory; all files are read |
| Host Key Documents (HKDs) | `hkds/` | Directory; all files are read |
| SE image header | `hdr/hdr.bin` | Binary file |
| Root CA (optional) | `root_ca.crt` | Single file |

Set `CERTS_OFFLINE_VERIFICATION=true` through `as.extraEnvVars` to verify the HKD
certificate chain offline. Do not set `SE_SKIP_CERTS_VERIFICATION=true` outside a
development environment because it disables HKD certificate chain verification.

```bash
# 1. Place all materials under a directory on the target s390x node, for example:
#    $IBM_SE_CREDS_DIR/{rsa/,certs/,crls/,hkds/,hdr/hdr.bin}
#    See deps/verifier/src/se/README.md for how to obtain the materials.

# 2. Install, pointing the chart at the node and directory:
helm upgrade --install trustee ./deployment/helm-chart \
  --namespace coco-trustee --create-namespace \
  -f ./deployment/helm-chart/scenarios/ibm-se.yaml \
  --set as.verifier.se.credsDir=$IBM_SE_CREDS_DIR \
  --set as.verifier.se.nodeName=<your-s390x-node-name>
```

Use an s390x AS image built with the `se-verifier` feature. Configure the image with
`as.image.repository` and `as.image.tag`; see `scenarios/ibm-se.yaml` for the complete
override. After installation, set the SE attestation policy as described in
`deps/verifier/src/se/README.md`.

## Validate the deployment

Inspect the deployed resources:

```bash
kubectl get deploy,pods,svc -n coco-trustee
helm status trustee -n coco-trustee
```

Render the chart without installing it:

```bash
helm dependency update ./deployment/helm-chart

helm template trustee ./deployment/helm-chart \
  -f ./deployment/helm-chart/scenarios/postgres-backend.yaml \
  --namespace coco-trustee > /tmp/trustee-render.yaml
```

{{% alert title="Note" color="info" %}}
If your cluster cannot resolve `*.svc.cluster.local` from Pods, set
`dnsHostAliasWorkaround: true` in your override values. After the Services exist, run
`helm upgrade` again so that Helm `lookup` can resolve the ClusterIPs.
{{% /alert %}}

Test KBS with `kbs-client`:

Build the client from the repository with `cargo build -p kbs-client --release`. When
ephemeral keys are enabled, the hook-created Secret whose name ends with
`bootstrap-user-keys` includes a pre-signed admin JWT under `KBS_ADMIN_TOKEN`. KBS
expects `authorization_mode = "AuthenticatedAuthorization"` and a bearer JWT with a
`role` claim that matches `[admin.authorization.regex_acl]`. The default role is
`admin`.

```bash
kubectl port-forward -n coco-trustee svc/trustee-kbs 8080:8080 &
SECRET=$(kubectl get secrets -n coco-trustee -o name | grep bootstrap-user-keys | head -1 | cut -d/ -f2)
kubectl get secret "$SECRET" -n coco-trustee -o jsonpath='{.data.KBS_ADMIN_TOKEN}' | base64 -d >/tmp/admin-token
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource-policy --allow-all
```

Set a confidential resource by configuring the admin token and then running
`set-resource`:

```bash
echo 'demo-payload' >/tmp/demo-resource.txt
kbs-client --url http://127.0.0.1:8080 config --admin-token-file /tmp/admin-token set-resource \
  --path my_repo/resource_type/demo --resource-file /tmp/demo-resource.txt
```

Fetch the resource by its KBS URI path. `get-resource` is a top-level subcommand that
uses the standard attestation and token flow for your client build and policy:

```bash
kbs-client --url http://127.0.0.1:8080 get-resource --path my_repo/resource_type/demo
```

## Uninstall

Remove the release:

```bash
helm uninstall trustee -n coco-trustee
```

{{% alert title="Note" color="info" %}}
When `secrets.useEphemeralGeneratedKeys` is `true` (default), a post-delete Helm hook automatically removes the release-scoped `*-bootstrap-user-keys` Secret.
{{% /alert %}}