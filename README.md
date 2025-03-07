# Dev Workspace Operator

Dev Workspace operator repository that contains the controller for the DevWorkspace Custom Resource. The Kubernetes API of the DevWorkspace is defined in the https://github.com/devfile/api repository.

## DevWorkspace CR

### Additional configuration

DevWorkspaces can be configured through DevWorkspace attributes and Kubernetes labels/annotations. For a list of all options available, see [additional documentation](docs/additional-configuration.adoc).

#### Restricted Access

The `controller.devfile.io/restricted-access` annotation specifies that a DevWorkspace needs additional access control (in addition to RBAC). When a DevWorkspace is created with the `controller.devfile.io/restricted-access` annotation set to `true`, the webhook server will guarantee
- Only the DevWorkspace Operator ServiceAccount or DevWorkspace creator can modify important fields in the DevWorkspace
- Only the DevWorkspace creator can create `pods/exec` into devworkspace-related containers.

This annotation should be used when a DevWorkspace is expected to contain sensitive information that should be protect above the protection provided by standard RBAC rules (e.g. if the DevWorkspace will store the user's OpenShift token in-memory).

Example:
```yaml
metadata:
  annotations:
    controller.devfile.io/restricted-access: true
```
## Prerequisites
- go 1.16 or later
- git
- sed
- jq
- yq (python-yq from https://github.com/kislyuk/yq#installation, other distributions may not work)
- skopeo (if building the OLM catalogsource)
- docker

Note: kustomize `v4.0.5` is required for most tasks. It is downloaded automatically to the `.kustomize` folder in this repo when required. This downloaded version is used regardless of whether or not kustomize is already installed on the system.

## Running the controller in a cluster

### With yaml resources

When deployed to Kubernetes, the controller requires [cert-manager](https://cert-manager.io) running in the cluster.
You can install it using `make install_cert_manager` if you don't run it already.
The minimum version of cert-manager is `v1.0.4`.

The controller can be deployed to a cluster provided you are logged in with cluster-admin credentials:

```bash
export DWO_IMG=quay.io/devfile/devworkspace-controller:next
make install
```

By default, the controller will expose workspace servers without any authentication; this is not advisable for public clusters, as any user could access the created workspace via URL.

See below for all environment variables used in the makefile.

> Note: The operator requires internet access from containers to work. By default, `crc setup` may not provision this, so it's necessary to configure DNS for Docker:
> ```
> # /etc/docker/daemon.json
> {
>   "dns": ["192.168.0.1"]
> }
> ```

### With OLM

DevWorkspace Operator has bundle and index images which allow to install it with OLM.
You need to register custom catalog source to make it available on your cluster with help:
```
DWO_INDEX_IMG=quay.io/devfile/devworkspace-operator-index:next
make register_catalogsource
```
After OLM finishes processing the created catalog source, DWO should appear on Operators page of OpenShift Console.

In order to build a custom bundle, the following env vars should be set:
| variable | purpose | default value |
|---|---|---|
| `DWO_BUNDLE_IMG` | Image used for Operator bundle image | `quay.io/devfile/devworkspace-operator-bundle:next` |
| `DWO_INDEX_IMG` | Image used for Operator index image | `quay.io/devfile/devworkspace-operator-index:next` |
| `DEFAULT_DWO_IMG` | Image used for controller when generating defaults | `quay.io/devfile/devworkspace-controller:next` |

To build the index image and register its catalogsource to the cluster, run
```
make generate_olm_bundle_yaml build_bundle_image build_index_image register_catalogsource
```

Note that setting `DEFAULT_DWO_IMG` while generating sources will result in local changes to the repo which should be `git restored` before committing. This can also be done by unsetting the `DEFAULT_DWO_IMG` env var and re-running `make generate_olm_bundle_yaml`

## Development

The repository contains a Makefile; building and deploying can be configured via the environment variables

|variable|purpose|default value|
|---|---|---|
| `DWO_IMG` | Image used for controller | `quay.io/devfile/devworkspace-controller:next` |
| `DEFAULT_DWO_IMG` | Image used for controller when generating default deployment templates. Can be used to override the controller image in the OLM bundle | `quay.io/devfile/devworkspace-controller:next` |
| `NAMESPACE` | Namespace to use for deploying controller | `devworkspace-controller` |
| `ROUTING_SUFFIX` | Cluster routing suffix (e.g. `$(minikube ip).nip.io`, `apps-crc.testing`). Required for Kubernetes | `192.168.99.100.nip.io` |
| `PULL_POLICY` | Image pull policy for controller | `Always` |
| `DEVWORKSPACE_API_VERSION` | Branch or tag of the github.com/devfile/api to depend on | `v1alpha1` |

Some of the rules supported by the makefile:

|rule|purpose|
|---|---|
| docker | build and push docker image |
| install | install controller to cluster |
| restart | restart cluster controller deployment |
| install_cert_manager | installs the cert-manager to the cluster (only required for Kubernetes) |
| uninstall | delete controller namespace `devworkspace-controller` and remove CRDs from cluster |
| help | print all rules and variables |

To see all rules supported by the makefile, run `make help`

### Test run controller
1. Take a look samples devworkspace configuration in `./samples` folder.
2. Apply any of them by executing `kubectl apply -f ./samples/flattened_theia-next.yaml -n <namespace>`
3. As soon as devworkspace is started you're able to get IDE url by executing `kubectl get devworkspace -n <namespace>`

### Run controller locally
```bash
export NAMESPACE="devworkspace-controller"
make install
oc patch deployment/devworkspace-controller-manager --patch "{\"spec\":{\"replicas\":0}}" -n $NAMESPACE
make run
```

When running locally, only a single namespace is watched; as a result, all devworkspaces have to be deployed to `${NAMESPACE}`

### Run controller locally and debug
Debugging the controller depends on [delve](https://github.com/go-delve/delve) being installed (`go install github.com/go-delve/delve/cmd/dlv@latest`). Note that `$GOPATH/bin` or `$GOBIN` must be added to `$PATH` in order for `make debug` to run correctly.

```bash
make install
oc patch deployment/devworkspace-controller-manager --patch "{\"spec\":{\"replicas\":0}}"
make debug
```

### Run webhook server locally and debug
Debugging the webhook server depends on `telepresence` being installed (`https://www.telepresence.io/docs/latest/install/`). Teleprescence works by redirecting traffic going from the webhook-server in the cluster to the local webhook-server you will be running on your computer.

```bash
make debug-webhook-server
```

when you are done debugging you have to manually uninstall the telepresence agent

```bash
make disconnect-debug-webhook-server
```

### Updating devfile API

[devfile API](https://github.com/devfile/api) is the Kube-native API for cloud development workspaces specification and the core dependency of the devworkspace-operator that should be regularly updated to the latest version. In order to do the update:

1. update `DEVWORKSPACE_API_VERSION` variable in the `Makefile` and `generate-deployment.sh`. The variable should correspond to the commit SHA from the [devfile API](https://github.com/devfile/api) repository
2. run the following scripts and the open pull request

```bash
make update_devworkspace_api update_devworkspace_crds # first commit
make generate manifests fmt generate_default_deployment generate_olm_bundle_yaml # second commit
```
Example of the devfile API update [PR](https://github.com/devfile/devworkspace-operator/pull/797)

### Controller configuration

Controller behavior can be configured using the `DevWorkspaceOperatorConfig` custom resource (`dwoc` for short). To configure the controller, create a `DevWorkspaceOperatorConfig` named `devworkspace-operator-config` in the same namespace as the controller. If using the Makefile to deploy the DevWorkspaceOperator, a pre-filled config is created automatically (see `deploy/default-config.yaml`).

Configuration settings in the `DevWorkspaceOperatorConfig` override default values found in [pkg/config](https://github.com/devfile/devworkspace-operator/tree/main/pkg/config). The only required configuration setting is `.routing.clusterHostSuffix`, which is required when running on Kubernetes.

To see documentation on configuration settings, including default values, use `kubectl explain` or `oc explain` -- e.g. `kubectl explain dwoc.config.routing.clusterHostSuffix`

### Remove controller from your K8s/OS Cluster
To uninstall the controller and associated CRDs, use the Makefile uninstall rule:
```bash
make uninstall
```
This will delete all custom resource definitions created for the controller, as well as the `devworkspace-controller` namespace.

### CI

#### GitHub actions

- [Next Dockerimage](https://github.com/devfile/devworkspace-operator/blob/main/.github/workflows/dockerimage-next.yml) action builds main branch and pushes it to [quay.io/devfile/devworkspace-controller:next](https://quay.io/repository/devfile/devworkspace-controller?tag=latest&tab=tags)
