k0s Control plane can be configured via a YAML config file. By default `k0s controller` command reads a file called `k0s.yaml` but can be told to read any yaml file via `--config` option.

## Configuration file reference

**Note:** Many of the options configure things deep down in the "stack" on various components. So please make sure you understand what is being configured and whether or not it works in your specific environment.

A full config file with defaults generated by the `k0s default-config` command:

```yaml
apiVersion: k0s.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s
spec:
  api:
    port: 6443
    k0sApiPort: 9443
    externalAddress: my-lb-address.example.com
    address: 192.168.68.104
    sans:
      - 192.168.68.104
  storage:
    type: etcd
    etcd:
      peerAddress: 192.168.68.104
  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12
    provider: calico
    calico:
      mode: vxlan
      vxlanPort: 4789
      vxlanVNI: 4096
      mtu: 0
      wireguard: false
      flexVolumeDriverPath: /usr/libexec/k0s/kubelet-plugins/volume/exec/nodeagent~uds
      withWindowsNodes: false
      overlay: Always
  podSecurityPolicy:
    defaultPolicy: 00-k0s-privileged
  telemetry:
    interval: 10m0s
    enabled: true
  installConfig:
    users:
      etcdUser: etcd
      kineUser: kube-apiserver
      konnectivityUser: konnectivity-server
      kubeAPIserverUser: kube-apiserver
      kubeSchedulerUser: kube-scheduler
  images:
    konnectivity:
      image: us.gcr.io/k8s-artifacts-prod/kas-network-proxy/proxy-agent
      version: v0.0.13
    metricsserver:
      image: gcr.io/k8s-staging-metrics-server/metrics-server
      version: v0.3.7
    kubeproxy:
      image: k8s.gcr.io/kube-proxy
      version: v1.21.0
    coredns:
      image: docker.io/coredns/coredns
      version: 1.7.0
    calico:
      cni:
        image: calico/cni
        version: v3.16.2
      flexvolume:
        image: calico/pod2daemon-flexvol
        version: v3.16.2
      node:
        image: calico/node
        version: v3.16.2
      kubecontrollers:
        image: calico/kube-controllers
        version: v3.16.2
  konnectivity:
    agentPort: 8132
    adminPort: 8133
```

### `spec.api`

- `externalAddress`: If k0s controllers are running behind a loadbalancer provide the loadbalancer address here. This will configure all cluster components to connect to this address and also configures this address to be used when joining new nodes into the cluster.
- `address`: The local address to bind API on. Also used as one of the addresses pushed on the k0s create service certificate on the API. Defaults to first non-local address found on the node.
- `sans`: List of additional addresses to push to API servers serving certificate
- `extraArgs`: Map of key-values (strings) for any extra arguments you wish to pass down to Kubernetes api-server process
- `port`: custom port for kube-api server to listen on (default value 6443)
- `k0sApiPort`: custom port for k0s-api server to listen on (default value 9443)

Keep in mind, in case if `port` and `k0sApiPort` are used with `externalAddress` setting, the LB serving at `externalAddress` must listen on the same ports.  

### `spec.controllerManager`

- `extraArgs`: Map of key-values (strings) for any extra arguments you wish to pass down to Kubernetes controller manager process

### `spec.scheduler`

- `extraArgs`: Map of key-values (strings) for any extra arguments you wish to pass down to Kubernetes scheduler process

### `spec.storage`

- `type`: Type of the data store, either `etcd` or `kine`.
- `etcd.peerAddress`: Nodes address to be used for etcd cluster peering.
- `kine.dataSource`: [kine](https://github.com/rancher/kine/) datasource URL.

Using type `etcd` will make k0s to create and manage an elastic etcd cluster within the controller nodes.

### `spec.network`

- `provider`: Network provider, either `calico`, `kuberouter` or `custom`. In case of `custom` user can push any network provider. (default `kuberouter`)
- `podCIDR`: Pod network CIDR to be used in the cluster
- `serviceCIDR`: Network CIDR to be used for cluster VIP services.

**Note:** In case of custom network it's fully in users responsibility to configure ALL the CNI related setups. This includes the CNI provider itself plus all the host levels setups it might need such as CNI binaries.

**Note:** After the cluster has been initialized with one network provider, it is not currently supported to change the network provider.
#### `spec.network.calico`

- `mode`: `vxlan` (default) or `ipip`
- `vxlanPort`: The UDP port to use for VXLAN (default `4789`)
- `vxlanVNI`: The virtual network ID to use for VXLAN. (default: `4096`)
- `mtu`: MTU to use for overlay network (default `0` which makes calico detect optimal MTU during bootstrap)
- `wireguard`: enable wireguard based encryption (default `false`). Your host system must be wireguard ready. See https://docs.projectcalico.org/security/encrypt-cluster-pod-traffic for details.
- `flexVolumeDriverPath`: The host path to use for Calicos flex-volume-driver (default: `/usr/libexec/k0s/kubelet-plugins/volume/exec/nodeagent~uds`). This should only need to be changed if the default path is unwriteable. See https://github.com/projectcalico/calico/issues/2712 for details. This option should ideally be paired with a custom volumePluginDir in the profile used on your worker nodes.
- `ipAutodetectionMethod`: To force non-default behaviour for Calico to pick up the interface for pod network inter-node routing. (default `""`, i.e. not set so Calico will use it's own defaults) See more at: https://docs.projectcalico.org/reference/node/configuration#ip-autodetection-methods

#### `spec.network.kuberouter`

- `autoMTU`: Autodetection of used MTU (default `true`)
- `mtu`: Override MTU setting, if set `autoMTU` needs to be set to `false`
- `peerRouterIPs`: comma separated list of [global peer addresses](https://github.com/cloudnativelabs/kube-router/blob/master/docs/bgp.md#global-external-bgp-peers)
- `peerRouterASNs`: comma separated list of [global peer ASNs](https://github.com/cloudnativelabs/kube-router/blob/master/docs/bgp.md#global-external-bgp-peers)

Kube-router allows many aspects of the networking to be configured per node, service and pod. For further details see [kube-router user guide](https://github.com/cloudnativelabs/kube-router/blob/master/docs/user-guide.md)

### `spec.podSecurityPolicy`

Configures the default [psp](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) to be set. k0s creates two PSPs out of box:

- `00-k0s-privileged` (default): no restrictions, always also used for Kubernetes/k0s level system pods
- `99-k0s-restricted`: no host namespaces or root users allowed, no bind mounts from host

As a user you can of course create any supplemental PSPs and bind them to users / access accounts as you need.

### `spec.workerProfiles`

Array of `spec.workerProfiles.workerProfile`
Each element has following properties:
- `name`: string, name, used as profile selector for the worker process
- `values`: mapping object

For each profile the control plane will create separate ConfigMap with kubelet-config yaml.
Based on the `--profile` argument given to the `k0s worker` the corresponding ConfigMap would be used to extract `kubelet-config.yaml` from.
`values` are recursively merged with default `kubelet-config.yaml`

There are a few fields that cannot be overridden:
- `clusterDNS`
- `clusterDomain`
- `apiVersion`
- `kind`

Example:

```
spec:
  workerProfiles:
    - name: custom-role
      values:
         key: value
         mapping:
             innerKey: innerValue
```


Custom volumePluginDir:

```
spec:
  workerProfiles:
    - name: custom-role
      values:
         volumePluginDir: /var/libexec/k0s/kubelet-plugins/volume/exec
```

### `spec.images`
Each node under the `images` key has the same structure
```
spec:
  images:
    konnectivity:
      image: calico/kube-controllers
      version: v3.16.2
```

Following keys are available:

- `spec.images.konnectivity`
- `spec.images.metricsserver`
- `spec.images.kubeproxy`
- `spec.images.coredns`
- `spec.images.calico.cni`
- `spec.images.calico.flexvolume`
- `spec.images.calico.node`
- `spec.images.calico.kubecontrollers`
- `spec.images.repository`

If `spec.images.repository` is set and not empty, every image will be pulled from `images.repository`
If `spec.images.default_pull_policy` is set ant not empty, it will be used as a pull policy for each bundled image.

Example:
```
images:
  repository: "my.own.repo"
  konnectivity:
    image: calico/kube-controllers
    version: v3.16.2
  metricsserver:
    image: gcr.io/k8s-staging-metrics-server/metrics-server
    version: v0.3.7
```

In the runtime the image names will be calculated as `my.own.repo/calico/kube-controllers:v3.16.2` and `my.own.repo/k8s-staging-metrics-server/metrics-server`.

This only affects the location where images are getting pulled, omitting an image specification here will not disable the component from being deployed.

### `spec.extensions.helm`

List of [Helm](https://helm.sh) repositories and charts to deploy during cluster bootstrap. For more information, see [Helm Charts](helm-charts.md).

### `spec.konnectivity`

Konnectivity related settings

- `agentPort` agent port to listen on (default 8132)
- `adminPort` admin port to listen on (default 8133)
### Telemetry

To build better end user experience we collect and send telemetry data from clusters. It is enabled by default and can be disabled by settings corresponding option as `false`
The default interval is 10 minutes, any valid value for `time.Duration` string representation can be used as a value.
Example
```
spec:
    telemetry:
      interval: 2m0s
      enabled: true
```



## Configuration Validation

k0s command-line interface has the ability to validate config syntax:

```
$ k0s validate config --config path/to/config/file
```

`validate config` sub-command can validate the following:

1. YAML formatting
2. [SANs addresses](#specapi-1)
3. [Network providers](#specnetwork-1)
4. [Worker profiles](#specworkerprofiles) 