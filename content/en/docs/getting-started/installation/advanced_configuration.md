---
title: Customization
description: Customize the Helm chart deployment of Confidential Containers
weight: 1
categories:
- installation
- helm
---
The Helm chart can be customized by passing additional parameters to the `helm install` command.

## Important Notes

1. **Node Selectors:** When setting node selectors with dots in the key, escape them: `node-role\.kubernetes\.io/worker`
2. **Namespace:** All examples use `coco-system` namespace. Adjust as needed for your environment
3. **Architecture:** The default architecture is x86_64. Other architectures must be explicitly specified
4. **Comma Escaping:** When using `--set` with values containing commas, escape them with `\,`

## Customizing deployment

You can combine architecture values files (with `-f`) and/or with `--set` flags for customizations.


### Using `--set` flags

To customize the installation using `--set` flags, run one of the following commands based on your architecture:

```bash
# For x86_64

helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  --set kata-as-coco-runtime.debug=true \
  --namespace coco-system \
  --create-namespace

# For s390x

helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  -f https://raw.githubusercontent.com/confidential-containers/charts/main/values/kata-s390x.yaml \
  --set kata-as-coco-runtime.debug=true \
  --namespace coco-system \
  --create-namespace
```

Parameters that are commonly customized (use `--set` flags):

| Parameter                               | Description                                             | Default  |
|-----------------------------------------|---------------------------------------------------------|----------|
| `kata-as-coco-runtime.imagePullPolicy`  | Image pull policy                                       | `Always` |
| `kata-as-coco-runtime.imagePullSecrets` | Image pull secrets for private registry                 | `[]`     |
| `kata-as-coco-runtime.k8sDistribution`  | Kubernetes distribution (k8s, k3s, rke2, k0s, microk8s) | `k8s`    |
| `kata-as-coco-runtime.nodeSelector`     | Node selector for deployment                            | `{}`     |
| `kata-as-coco-runtime.debug`            | Enable debug logging                                    | `false`  |

#### Structured Configuration (Kata Containers)

The chart uses Kata Containers' structured configuration format for TEE shims. Parameters set by architecture-specific
kata runtime values files:

| Parameter                                                          | Description                                                                             | Set by values/kata-*.yaml |
|--------------------------------------------------------------------|-----------------------------------------------------------------------------------------|---------------------------|
| `architecture`                                                     | Architecture label for NOTES                                                            | `x86_64` or `s390x`       |
| `kata-as-coco-runtime.snapshotter.setup`                           | Array of snapshotters to set up (e.g., `["nydus"]`)                                     | Architecture-specific     |
| `kata-as-coco-runtime.shims.<shim-name>.enabled`                   | Enable/disable specific shim (e.g., `qemu-snp`, `qemu-tdx`, `qemu-se`, `qemu-coco-dev`) | Architecture-specific     |
| `kata-as-coco-runtime.shims.<shim-name>.supportedArches`           | List of architectures supported by the shim                                             | Architecture-specific     |
| `kata-as-coco-runtime.shims.<shim-name>.containerd.snapshotter`    | Snapshotter to use for containerd (e.g., `nydus`, `""` for none)                        | Architecture-specific     |
| `kata-as-coco-runtime.shims.<shim-name>.containerd.forceGuestPull` | Enable experimental force guest pull                                                    | `false`                   |
| `kata-as-coco-runtime.shims.<shim-name>.crio.guestPull`            | Enable guest pull for CRI-O                                                             | Architecture-specific     |
| `kata-as-coco-runtime.shims.<shim-name>.agent.httpsProxy`          | HTTPS proxy for guest agent                                                             | `""`                      |
| `kata-as-coco-runtime.shims.<shim-name>.agent.noProxy`             | No proxy settings for guest agent                                                       | `""`                      |
| `kata-as-coco-runtime.runtimeClasses.enabled`                      | Create runtimeclass resources                                                           | `true`                    |
| `kata-as-coco-runtime.runtimeClasses.createDefault`                | Create default k8s runtimeclass                                                         | `false`                   |
| `kata-as-coco-runtime.runtimeClasses.defaultName`                  | Name for default runtimeclass                                                           | `"kata"`                  |
| `kata-as-coco-runtime.defaultShim.<arch>`                          | Default shim per architecture (e.g., `amd64: qemu-snp`)                                 | Architecture-specific     |

#### Additional Parameters (kata-deploy options)

These inherit from kata-deploy defaults but can be overridden:

| Parameter                                     | Description                       | Default                               |
|-----------------------------------------------|-----------------------------------|---------------------------------------|
| `kata-as-coco-runtime.image.reference`        | Kata deploy image                 | `quay.io/kata-containers/kata-deploy` |
| `kata-as-coco-runtime.image.tag`              | Kata deploy image tag             | Chart's application version           |
| `kata-as-coco-runtime.env.installationPrefix` | Installation path prefix          | `""` (uses kata-deploy defaults)      |
| `kata-as-coco-runtime.env.multiInstallSuffix` | Suffix for multiple installations | `""`                                  |

See [quickstart](https://github.com/confidential-containers/charts/blob/main/QUICKSTART.md) for complete customization examples and usage.
 
### Using file based values

Prepare `my-values.yaml` file in one of the following ways:

- Using latest default values downloaded from the chart:

  ```bash
  helm show values oci://ghcr.io/confidential-containers/charts/confidential-containers > my-values.yaml
  ```

- Using newly created file `my-values.yaml` with your customizations, e.g., for s390x with debug and node selector:

  ```yaml
  architecture: s390x
  
  kata-as-coco-runtime:
    env:
      debug: "true"
      shims: "qemu-coco-dev qemu-se"
      snapshotterHandlerMapping: "qemu-coco-dev:nydus,qemu-se:nydus"
      agentHttpsProxy: "http://proxy.example.com:8080"
    nodeSelector:
      node-role.kubernetes.io/worker: ""
  ```

  List of custom values examples can be found in the [examples-custom-values](https://github.com/confidential-containers/charts/blob/main/examples-custom-values.yaml).

Install chart using your custom values file:

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  -f my-values.yaml \
  --namespace coco-system \
  --create-namespace
```

#### Multiple combined customization options

Customizations using `--set` flags can be combined with file based values using `-f`.

See below example which will provide s390x architecture, enable debug logging, and set a node selector for worker nodes.

```bash
helm install coco oci://ghcr.io/confidential-containers/charts/confidential-containers \
  -f https://raw.githubusercontent.com/confidential-containers/charts/main/values/kata-s390x.yaml \
  --set kata-as-coco-runtime.env.debug=true \
  --set kata-as-coco-runtime.nodeSelector."node-role\.kubernetes\.io/worker"="" \
  --set kata-as-coco-runtime.k8sDistribution=k3s \
  --namespace coco-system \
  --create-namespace
```