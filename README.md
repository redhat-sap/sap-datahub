# OpenShift templates and utilities for SAP Data Hub

Currently contains the following OpenShift templates:

- [`sdh-observer.yaml`](./sdh-observer.yaml)

## Observer for SAP Data Hub

SAP Data Hub deployment resources require certain modifications in order to run on OpenShift. Those are at least the following:

- `statefulset/vsystem-vrep` must mount `emptyDir` volume on `/exports` directory on RHCOS nodes
- `deployment/vflow-*` must be run as `spc_t` if it mounts `/var/run/docker.sock` socket from the host
- `deployment/vflow-*` must be run with insecure registry parameter if the vflow registry is insecure
- `vsystem-iptables` containers must run as *privileged* on RHCOS nodes unless all the required kernel modules are pre-loaded
- `diagnostics-fluentd` pods need access to `/var/log` directories on nodes

Once the template is deployed, the `vflow-observer` pod will modify all the mentioned SDH resources running in `SDH_NAMESPACE` that require modification. That will terminate existing instances (pods) and spawn new ones. 

### Modifications in detail

#### vsystem-vrep modification

`vsystem-vrep` `statefulset` will be patched to mount `emptyDir` volume at `/exports` directory in order to enable NFS exports in the container running on top overlayfs which is the default filesystem in Red Hat Enterprise Linux CoreOS (RHCOS).

This is applicable only to OpenShift cluster 4.1 and newer. The resource running on earlier releases will not be modified.

#### Running SDH Pipeline Modeler as a Super Privileged Container

The Pipeline Modeler needs to access `/var/run/docker.sock` socket on the nodes if run without kaniko image builder (enabled with `--enable-kaniko=yes` parameter passed to the SDH installer). The access is, however, blocked by the default SELinux policy. In order to allow Pipeline Modeler to access the socket, it must be run as Super Privileged Container - it must be run in`spc_t` domain.

The `sdh-observer` will patch any `vflow` deployments that attempt to mount the `docker.sock` on the host.

**NOTE**: The recommended and secure approach is to enable kaniko image builds instead. The kaniko feature is available starting from SDH release 2.5.

#### SDH Pipeline Modeler with insecure Docker registry

In SAP Data Hub 2.5 and 2.6 releases, it is possible to configure insecure registry for Pipeline Modeler (aka vflow pod) neither via installer nor in the UI.

The insecure registry needs to be set if the container registry listens on insecure port (HTTP) or the communication is encrypted using a self-signed certificate.

Without the insecure registry set, kaniko builder cannot push built images into the configured registry for the Pipeline Modeler (see "Container Registry for Pipeline Modeler" Input Parameter at [the official SAP Data Hub documentation]( https://help.sap.com/viewer/e66c399612e84a83a8abe97c0eeb443a/2.5.latest/en-US/abfa9c73f7704de2907ea7ff65e7a20a.html).

The registry to mark as insecure will be determined from the `installer-config` secret located in the `SDH_NAMESPACE`. If another registry shall be marked as insecure, it can be specified with an additional `REGISTRY` parameter.

Please refer to the [SAP Data Hub 2.6 on OpenShift Container Platform 4](https://access.redhat.com/articles/4324391) for more information.

Important environment variables:

- `MARK_REGISTRY_INSECURE` - must be set to `true` if the registry shall be marked as insecure. The default is `false`.
- `REGISTRY` - can be specified to mark particular registry as insecure. The default is to determine the registry from the `installer-config` secret located in the `SDH_NAMESPACE`.

#### `vsystem-iptables` containers running as *privileged*

The `vsystem-iptables` containers need iptables and related kernel modules loaded on the nodes in order to manipulate firewall. On Red Hat Enterprise Linux CoreOS, nftables are used instead of iptables. Therefor, these modules are not loaded by default. Unless the modules are pre-loaded on the worker nodes using [MachineConfig API](https://github.com/openshift/machine-config-operator/tree/master-4.1#applying-configuration-changes-to-the-cluster), the `vsystem-app` deployments including these containers will fail to run, rendering the SAP Data Hub unusable.

The `sdh-observer` allows to patch such deployments to make the `vsystem-iptables` containers `privileged`. This allows them to load all the kernel modules they need.

The template spawns a pod that observes the particular namespace where SAP Data Hub runs and marks each `"vsystem-iptables"` container in all `"vsystem-app"` deployments as *privileged*. The pod will keep monitoring the namespace and patching the deployments as soon as they appear.

This is applicable only to OpenShift cluster 4.1 and newer. The resources running on earlier releases will not be modified.

In order to allow this functionality, the `sdh-observer` template needs to be run with parameter `MAKE_VSYSTEM_IPTABLES_PODS_PRIVILEGED=true` like this:

    oc process -f https://raw.githubusercontent.com/miminar/sdh-helpers/master/sdh-observer.yaml \
        NAMESPACE=$SDH_NAMESPACE MAKE_VSYSTEM_IPTABLES_PODS_PRIVILEGED=true | oc create -f -

However, it is recommended to [pre-load the modules](https://access.redhat.com/articles/4324391#preload-kernel-modules-post) on the worker nodes instead.

### Usage

The template must be instantiated before the SAP Data Hub's installation. Both the target namespace and the namespace where the SAP Data Hub will be installed must exist before the instantiation. 

Targeted at:

- OpenShift 3.10 or higher
- SAP Data Hub is 2.5 or higher

**NOTE**: you will likely see the very first deployment of license-management to fail during the SAP Data Hub's installation with a message like the following:

```
2019-07-24T07:28:54+0000 [INFO] Solutions were successfully imported!
2019-07-24T07:28:54+0000 [INFO] Initializing system tenant...
2019-07-24T07:28:54+0000 [INFO] Initializing License Manager in system tenant...2019-07-24T07:29:35+0000 [ERROR] Couldn't start License Manager!
The response: Error: http status code 502 Bad Gateway (502)
2019-07-24T07:29:35+0000 [ERROR] Failed to initialize vSystem, will retry in 30 sec...
2019-07-24T07:30:05+0000 [INFO] Wait until vora cluster is ready...
```

The error is harmless and can be ignored. The next deployment will succeed.

#### Important template parameters

- `NAMESPACE` - (**mandatory**) the project name, where the template shall be instantiated
- `BASE_IMAGE_TAG` - (**mandatory**) must correspond to the OpenShift cluster's release (e.g. `v3.11` or `4.1`)
- `MARK_REGISTRY_INSECURE` - shall be set to `true` if an insecure registry is used
- `MAKE_VSYSTEM_IPTABLES_PODS_PRIVILEGED` - on OCP 4, when the needed [kernel modules were not preloaded](https://access.redhat.com/articles/4324391#preload-kernel-modules-post), this must be set to `true` to allow the `vsystem-iptables` containers to load the modules on their own

##### Determining BASE_IMAGE_TAG

The `BASE_IMAGE_TAG` determines the version OpenShift client binary to use to talk to cluster. Although the difference between client and server minor release can be 1, it is recommended to use the same client version.

Prerequisites:

- installed `skopeo`, `jq` and `atomic-openshift-clients` (OCP 3.11) or `openshift-clients` (OCP 4.1 or higher) RPM packages
- `bash` shell
- logged in to OCP cluster

To determine the value of `BASE_IMAGE_TAG`, execute the following commands (note, you must be logged in to O:

    ver="$(oc version | gawk '/^(openshift|[Ss]erver [Vv]ersion)/{
        match($0,/.*v([0-9]+)\.([0-9]+)\./,a); if (a[1] == 1 && a[2] > 11) {
                major=4; minor=a[2] - 12
            } else {
                major=a[1]; minor=a[2]
            }; { printf ("%s.%s\n", major, minor) }
        }')"
    skopeo inspect docker://quay.io/openshift/origin-cli:latest | jq -r '.RepoTags[] | select(match("^v?'"$ver"'"))' | head -n 1

That will select the latest version of `quay.io/openshift/origin-cli` image matching the OpenShift server's minor release.

For example, command will return `v3.11` for OCP 3.11 server release and `4.1` for OCP server 4.1 release.

#### Deploying in the Data Hub project

If running the observer in the same namespace/project as Data Hub, instantiate the template as is in the desired namespace:

    oc project $SDH_NAMESPACE
    oc process -f https://raw.githubusercontent.com/miminar/sdh-helpers/master/sdh-observer.yaml \
        NAMESPACE=$SDH_NAMESPACE MARK_REGISTRY_INSECURE=true | oc create -f -

#### Deploying in a different project

If running in a different/new namespace/project, instantiate the template with parameters `SDH_NAMESPACE` and `NAMESPACE`, e.g.:

    SDH_NAMESPACE=sdh26                 # where SDH will be installed
    NAMESPACE=sapdatahub-admin          # where the contents of this template will be instantiated
    oc new-project $SDH_NAMESPACE
    oc new-project $NAMESPACE
    oc process -f https://raw.githubusercontent.com/miminar/sdh-helpers/master/sdh-observer.yaml \
        SDH_NAMESPACE=$SDH_NAMESPACE NAMESPACE=$NAMESPACE MARK_REGISTRY_INSECURE=true | oc create -f -
