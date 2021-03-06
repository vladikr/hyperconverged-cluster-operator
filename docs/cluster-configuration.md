# Cluster Configuration
## Introduction
The HyperConverged Cluster allows modifying the KubeVirt cluster configuration by editing the HyperConverged Cluster CR
(Custom Resource).

The HyperConverged Cluster operator copies the cluster configuration values to the other operand's CRs.

***Note***: The cluster configurations are supported only in API version `v1beta1` or higher.
## Infra and Workloads Configuration
Some configurations are done separately to Infra and Workloads. The CR's Spec object contains the `infra` and the 
`workloads` objects.

The structures of the `infra` and the `workloads` objects are the same. The HyperConverged Cluster operator will update 
the other operator CRs, according to the specific CR structure. The meaning is if, for example, the other CR does not 
support Infra cluster configurations, but only Workloads configurations, the HyperConverged Cluster operator will only
copy the Workloads configurations to this operator's CR.

Below are the cluster configuration details. Currently, only "Node Placement" configuration is supported.

### Node Placement
Kubernetes lets the cluster admin influence node placement in several ways, see 
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/ for a general overview.

The HyperConverged Cluster's CR is the single entry point to let the cluster admin influence the placement of all the pods directly and indirectly managed by the HyperConverged Cluster Operator.

The `nodePlacement` object is an optional field in the HyperConverged Cluster's CR, under `spec.infra` and `spec.workloads`
fields.

***Note***: The HyperConverged Cluster operator does not allow modifying of the workloads' node placement configurations if there are already
existing virtual machines or data volumes. 

The `nodePlacement` object contains the following fields:
* `nodeSelector` is the node selector applied to the relevant kind of pods. It specifies a map of key-value pairs: for 
the pod to be eligible to run on a node,	the node must have each of the indicated key-value pairs as labels 	(it can
have additional labels as well). See https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector.
* `affinity` enables pod affinity/anti-affinity placement expanding the types of constraints
that can be expressed with nodeSelector.
affinity is going to be applied to the relevant kind of pods in parallel with nodeSelector
See https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity.
* `tolerations` is a list of tolerations applied to the relevant kind of pods.
See https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/ for more info.

#### Operators placement
The HyperConverged Cluster Operator and the operators for its component are supposed to be deployed by the Operator Lifecycle Manager (OLM).
Thus, the HyperConverged Cluster Operator is not going to directly influence its own placement but that should be influenced by the OLM.
The cluster admin indeed is allowed to influence the placement of the Pods directly created by the OLM configuring a [nodeSelector](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/subscription-config.md#nodeselector) or [tolerations](https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/subscription-config.md#tolerations) directly on the OLM subscription object.

#### Node Placement Examples
* Place the infra resources on nodes labeled with "nodeType = infra", and workloads in nodes labeled with "nodeType = nested-virtualization", using node selector:
  ```yaml
  ...
  spec:
    infra:
      nodePlacement:
        nodeSelector:
          nodeType: infra
    workloads:
      nodePlacement:
        nodeSelector:
          nodeType: nested-virtualization
  ```
* Place the infra resources on nodes labeled with "nodeType = infra", and workloads in nodes labeled with 
"nodeType = nested-virtualization", preferring nodes with more than 8 CPUs, using affinity:
  ```yaml
  ...
  spec:
    infra:
      nodePlacement:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: nodeType
                  operator: In
                  values:
                  - infra
    workloads:
      nodePlacement:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: nodeType
                  operator: In
                  values:
                  - nested-virtualization
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                - key: my-cloud.io/num-cpus
                  operator: gt
                  values:
                  - 8
  ```
* In this example, there are several nodes that are saved for KubeVirt resources (e.g. VMs), already set with the 
`key=kubevirt:NoSchedule` taint. This taint will prevent any scheduling to these nodes, except for pods with the matching 
tolerations.
  ```yaml
  ...
  spec:
    workloads:
      nodePlacement:
        tolerations:
        - key: "key"
          operator: "Equal"
          value: "kubevirt"
          effect: "NoSchedule"
  ```




## FeatureGates
The `featureGates` field is an optional set of optional boolean feature enabler. The features in this list are advanced 
or new features that are not enabled by default.

To enable a feature, add its name to the `featureGates` list and set it to `true`. Missing or `false` feature gates 
disables the feature.

### hotplugVolumes Feature Gate
Set the `hotplugVolumes` feature gate in order to allow attaching a data volume to a running VMI.

### withHostModelCPU Feature Gate
Set the `withHostModelCPU`  feature gate in order to enable support migration for VMs with host-model CPU mode

Additional information: [LibvirtXMLCPUModel](https://wiki.openstack.org/wiki/LibvirtXMLCPUModel)

### withHostPassthroughCPU Feature Gate
Set the `withHostPassthroughCPU`  feature gate in order to allow migrating a virtual machine with CPU host-passthrough mode. 

Additional information: [LibvirtXMLCPUModel](https://wiki.openstack.org/wiki/LibvirtXMLCPUModel)

**note**: This should be enabled only when the Cluster is homogeneous from CPU HW perspective doc here

### hypervStrictCheck Feature Gate
Set the `hypervStrictCheck` feature gate in order to enable [HyperV enlightenments](https://kubevirt.io/user-guide/#/creation/guest-operating-system-information?id=hyperv-optimizations) for Kubevirt.

**Default: `true`**

To override the default, specify the featureGate in the HCO configuration.

### Feature Gates Example

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
spec:
  infra: {}
  workloads: {}
  featureGates:
    hotplugVolumes: true
    withHostModelCPU: true
    withHostPassthroughCPU: true
```

## Configurations via Annotations
In addition to `featureGates` field in HyperConverged CR's spec, the user can set annotations in the HyperConverged CR to unfold more configuration options.  
**Warning:** Annotations are less formal means of cluster configuration and may be dropped without the same deprecation process of a regular API, such as in the `spec` section.

### OvS Opt-In Annotation
Starting from HCO version 1.3.0, OvS CNI support is disabled by default on new installations.  
In order to enable the deployment of OvS CNI DaemonSet on all _workload_ nodes, an annotation of `deployOVS: true` must be set on HyperConverged CR.  
It can be set while creating the HyperConverged custom resource during the initial deployment, or during run time.
* To enable OvS CNI on the cluster, the HyperConverged CR should be similar to:  
```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  annotations:
    deployOVS: "true"
...
```
* OvS CNI can also be enabled during run time of HCO, by annotating its CR:
```
kubectl annotate HyperConverged kubevirt-hyperconverged -n kubevirt-hyperconverged deployOVS=true --overwrite
```

If HCO was upgraded to 1.3.0 from a previous version, the annotation will be added as `true` and OvS will be deployed.  
Subsequent upgrades to newer versions will preserve the state from previous version, i.e. OvS will be deployed in the upgraded version if and only if it was deployed in the previous one.

### jsonpatch Annotations
HCO enables users to modify the operand CRs directly using jsonpatch annotations in HyperConverged CR.  
Modifications done to CRs using jsonpatch annotations won't be reconciled back by HCO to the opinionated defaults.  
The following annotations are supported in the HyperConverged CR:
* `kubevirt.kubevirt.io/jsonpatch` - for KubeVirt configurations
* `containerizeddataimporter.kubevirt.io/jsonpatch` - for CDI configurations
* `networkaddonsconfig.kubevirt.io/jsonpatch` - for CNAO configurations

The content of the annotation will be a json array of patch objects, as defined in [RFC6902](https://tools.ietf.org/html/rfc6902).
The patch’s path is relative to the `spec` field in each CR.

#### Examples
* The user wants to set the KubeVirt CR’s `spec.configuration.migrations.allowPostCopy` field to `true`. In order to do that, the following annotation should be added to the HyperConverged CR:
```yaml
metadata:
  annotations:
    kubevirt.kubevirt.io/jsonpatch: |-
      [
        {
          "op": "add",
          "path": "/configuration/migrations",
          "value": '{"allowPostCopy": "true"}'
        }
      ]
```
* The user wants to override the default URL used when uploading to a DataVolume, by setting the CDI CR's `spec.config.uploadProxyURLOverride` to `myproxy.example.com`. In order to do that, the following annotation should be added to the HyperConverged CR:
```yaml
metadata:
  annotations:
    containerizeddataimporter.kubevirt.io/jsonpatch: |-
      [
        {
          "op": "add",
          "path": "/config/uploadProxyURLOverride",
          "value": "myproxy.example.com"
        }
      ]
```

**_Note:_** The full configurations options for Kubevirt, CDI and CNAO which are available on the cluster, can be explored by using `kubectl explain <resource name>.spec`. For example:  
```yaml
$ kubectl explain kv.spec
KIND:     KubeVirt
VERSION:  kubevirt.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     <empty>

FIELDS:
   certificateRotateStrategy	<Object>

   configuration	<Object>
     holds kubevirt configurations. same as the virt-configMap

   customizeComponents	<Object>

   imagePullPolicy	<string>
     The ImagePullPolicy to use.

   imageRegistry	<string>
     The image registry to pull the container images from Defaults to the same
     registry the operator's container image is pulled from.
  
  <truncated>
```

To inspect lower-level objects onder `spec`, they can be specified in `kubectl explain`, recursively. e.g.  
```yaml
$ kubectl explain kv.spec.configuration.network
KIND:     KubeVirt
VERSION:  kubevirt.io/v1

RESOURCE: network <Object>

DESCRIPTION:
     NetworkConfiguration holds network options

FIELDS:
   defaultNetworkInterface	<string>

   permitBridgeInterfaceOnPodNetwork	<boolean>

   permitSlirpInterface	<boolean>
```

* To explore kubevirt configuration options, use `kubectl explain kv.spec`
* To explore CDI configuration options, use `kubectl explain cdi.spec`
* To explore CNAO configuration options, use `kubectl explain networkaddonsconfig.spec`

### WARNING
Using the jsonpatch annotation feature incorrectly might lead to unexpected results and could potentially render the Kubevirt-Hyperconverged system unstable.  
The jsonpatch annotation feature is particularly dangerous when upgrading Kubevirt-Hyperconverged, as the structure or the semantics of the underlying components' CR might be changed. Please remove any jsonpatch annotation usage prior the upgrade, to avoid any potential issues.
**USE WITH CAUTION!**
