---
title: "SDN for all the networks! Configuring underlay connectivity"
date: 2023-06-12
categories:
  - blog
tags:
  - Kubernetes
  - KubeVirt
  - CNI
  - SDN
  - NMState
---

In this post I will show and explain how to configure secondary networks in
OVN-Kubernetes connected to the physical underlay of the cluster. The physical
underlay will be configured using
[Kubernetes-NMState](https://github.com/nmstate/kubernetes-nmstate).

This blog post assumes you have the following software installed in your
Kubernetes cluster:
- KubeVirt
- OVN-Kubernetes
- Kubernetes-NMState

KubeVirt will allow you to run virtualized workloads alongside pods in the same
cluster, OVN-Kubernetes will be the CNI implementing the network fabric for
both the default cluster, and secondary networks, while Kubernetes-NMState will
be responsible for configuring the physical networks in the Kubernetes nodes.

## Using a dedicated interface for the secondary networks
This option is available if you have more than one interface available in your
nodes.

Let's assume there is an extra interface available on each of the Kubernetes
nodes, and the network admin wants to use a single OVS bridge to connect
multiple physical networks - one on VLAN 10, another on VLAN 20. That scenario
is available by provisioning the following NNCP.
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-br1-multiple-networks
spec:
  desiredState:
    interfaces:
      - name: ovsbr1
        description: ovs bridge with eth1 as a port allowing multiple VLANs through
        type: ovs-bridge
        state: up
        bridge:
          options:
            stp: true
          port:
            - name: eth1
              vlan:
                mode: trunk
                trunk-tags:
                  - id: 10
                  - id: 20
    ovs-db:
      external_ids:
        ovn-bridge-mappings: "physnet:breth0,tenantblue:ovsbr1,tenantred:ovsbr1"
```

Using the `NodeNetworkConfigurationPolicy` - a Kubernetes-NMState CRD - we are
requesting an OVS bridge - named `ovsbr1` - with `STP` enabled (Spanning tree
protocol).

The interface `eth1` is configured as a port of the aforementioned OVS bridge,
and VLANs 10, and 20 are allowed in that port. Also, VLAN 0 is applied to
untagged traffic, as per the
[Open vSwitch documentation](https://docs.openvswitch.org/en/latest/ref/ovs-actions.7/#the-ovs-normal-pipeline).

Finally, the
[OVN bridge mapping](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/networking_guide/bridge-mappings)
API is configured to specify which OVS bridges are used for each physical network.
It creates a patch port between the OVN integration bridge - `br-int` - and the
OVS bridge you tell it to, and traffic will be forwarded to/from it with the
help of a
[localnet port](https://man7.org/linux/man-pages/man5/ovn-nb.5.html#Logical_Switch_Port_TABLE).

Referring to the NNCP shown above, we are configuring three physical networks:
- physnet: cluster's default north/south network. Attached to `br-ex`.
- tenantblue: a network whose config will be shown. Attached to `ovsbr1`.
- tenantred: a network whose config we will show. Attached to `ovsbr1`.

After provisioning it, the network admin should ensure the policy was applied:
```yaml
kubectl get nncp ovs-br1-multiple-networks
NAME              STATUS      REASON
br-ex-multi-net   Available   SuccessfullyConfigured
```

### Configuring the Gateway
I am assuming you're using a virtualized infrastructure - i.e. your Kubernetes
nodes are VMs running on a bare-metal machine. On my environment, those look
like:
```bash
virsh list 
 Id   Name                      State
-----------------------------------------
 24   multi-homing-ctlplane-0   running
 25   multi-homing-ctlplane-1   running
 26   multi-homing-ctlplane-2   running
 27   multi-homing-worker-0     running
 28   multi-homing-worker-1     running
```

Quite probably your VMs will be interconnected via a linux (or OVS ...) bridge.
On my environment, those look like:
```bash
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 34:48:ed:f3:f6:b8 brd ff:ff:ff:ff:ff:ff
    altname enp24s0f0
    inet 10.46.41.66/24 brd 10.46.41.255 scope global dynamic noprefixroute eno1
       valid_lft 40430sec preferred_lft 40430sec
    inet6 2620:52:0:2e29:3648:edff:fef3:f6b8/64 scope global dynamic noprefixroute 
       valid_lft 2591976sec preferred_lft 604776sec
    inet6 fe80::3648:edff:fef3:f6b8/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
6: virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:9d:c5:32 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
65: secondary: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:9b:21:f1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.1/24 brd 192.168.123.255 scope global secondary
       valid_lft forever preferred_lft forever
```

The `secondary` linux bridge shown above (ifindex 65) will act as our gateway;
we will need to create a VLAN on top of it for each of the tags we have
configured on our NNCP:
```bash
ip link add link secondary name secondary.10 type vlan id 10
ip link add link secondary name secondary.20 type vlan id 20

ip addr add 192.168.180.1/24 dev secondary.10
ip addr add 192.168.185.1/24 dev secondary.20

ip link set secondary.10 up
ip link set secondary.20 up
```

### Defining the OVN-Kubernetes networks
Now that the physical underlay on the Kubernetes nodes is properly configured,
we can define the OVN-Kubernetes secondary networks. For that, we will use the
following `NetworkAttachmentDefinition` - a Multus-CNI CRD.

```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: tenantred
spec:
  config: |2
    {
            "cniVersion": "0.3.1",
            "name": "tenantred",
            "type": "ovn-k8s-cni-overlay",
            "topology": "localnet",
            "subnets": "192.168.180.0/24",
            "excludeSubnets": "192.168.180.1/32",
            "vlanID": 10,
            "netAttachDefName": "default/tenantred"
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: tenantblue
spec:
  config: |2
    {
            "cniVersion": "0.3.1",
            "name": "tenantblue",
            "type": "ovn-k8s-cni-overlay",
            "topology": "localnet",
            "subnets": "192.168.185.0/24",
            "excludeSubnets": "192.168.185.1/32",
            "vlanID": 20,
            "netAttachDefName": "default/tenantblue"
    }
```

As you can see above, we are defining two networks, whose names match the OVN
bridge mappings of the `NNCP`s defined in the previous section: `tenantblue`
and `tenantred`.

Also notice how each of those networks has a different VLAN, but both those
IDs are listed in the trunk configuration on the OVS bridge port - 10 and 20.

It is important to refer each of those networks is using a different subnet,
**and** we had to exclude the gateway IP from the range - otherwise,
OVN-Kubernetes would assign that IP address to the workloads.

### Defining the Virtual Machines
We will now define four Virtual Machines, all of them with a single network
interface. Two of those VMs will be connected to the `tenantblue` network,
while the other two will be connected to the `tenantred` network.

One of the VMs connected to each of the aforementioned networks will be
scheduled in each of the nodes - i.e. one `tenantblue` connected VM will be
scheduled in `multi-homing-worker-0`, and the other in `multi-homing-worker-1`.
The same is true for the `tenantred` connected VMs. A `nodeSelector` is used to
implement this.

```yaml
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-red-1
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-0
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: physnet-red
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: physnet-red
        multus:
          networkName: tenantred
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-red-2
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-1
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: flatl2-overlay
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: flatl2-overlay
        multus:
          networkName: tenantred
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-blue-1
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-0
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: physnet-blue
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: physnet-blue
        multus:
          networkName: tenantblue
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-blue-2
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-1
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: physnet-blue
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: physnet-blue
        multus:
          networkName: tenantblue
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
```

Now you can ping between the VMs in each network, plus access services on the
underlay, as long as they are reachable via the GW IP.

## Sharing a single physical interface
Another option - pretty much the only one available if your Kubernetes nodes
only feature a single physical interface - is to share that interface between
your cluster default and secondary networks.

For this, you should provision the following NNCP:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br-ex-multi-net
spec:
  desiredState:
    ovs-db:
      external_ids:
        ovn-bridge-mappings: "physnet:br-ex,externalvlan:br-ex,novlan:br-ex"
```

One thing to take into account is some platforms - i.e. Openshift - are really
conservative in regards to the interface managed by OVN-Kubernetes. You should
**not** - under any circumstance - try to reconfigure that interface.

Luckily, for Openvswitch, if you do not specify any list of trunks on an
interface it
[allows all VLANs through](https://docs.openvswitch.org/en/latest/ref/ovs-actions.7/#the-ovs-normal-pipeline),
thus, we can get away simply with specifying which physical networks can tap
the underlay.

In the following example, we are configuring a single physical network, but you
can add more. These can optionally encapsulate their traffic in VLANs.

### Defining the Gateway with a shared interface
The non-VLAN encapsulated traffic will traverse the trunk with VLAN tag 0,
and will be output from the OVS bridge without a tag, but you will need to
create a VLAN for the `externalvlan` network - let's say in uses VLAN ID 80.
Let's also assign it a subnet - `192.168.200.0/24` for instance.


```bash
ip link add link secondary name secondary.80 type vlan id 80

ip addr add 192.168.200.1/24 dev secondary.80

ip link set secondary.80 up
```

Finally, the linux bridge on my bare-metal node connected to the extra
interface on the Kubernetes nodes looks like:
```bash
ip a
...
65: secondary: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 52:54:00:9b:21:f1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.1/24 brd 192.168.123.255 scope global secondary
       valid_lft forever preferred_lft forever
...
```

We need to keep its subnet in mind for the next step.

### Defining the OVN-Kubernetes networks for shared bridge
These are the network definitions for the aforementioned networks:
```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: underlay
spec:
  config: |2
    {
            "cniVersion": "0.3.1",
            "name": "externalvlan",
            "type": "ovn-k8s-cni-overlay",
            "topology":"localnet",
            "vlanID": 80,
            "subnets": "192.168.200.0/24",
            "excludeSubnets": "192.168.200.1/32",
            "netAttachDefName": "default/underlay"
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: novlan
spec:
  config: |2
    {
            "cniVersion": "0.4.0",
            "name": "novlan",
            "type": "ovn-k8s-cni-overlay",
            "topology":"localnet",
            "subnets": "192.168.122.0/24",
            "excludeSubnets": "192.168.122.1/32",
            "netAttachDefName": "default/novlan"
    }
```

### Defining the Virtual Machines
We will now define four Virtual Machines, all of them with a single network
interface. Two of those VMs will be connected to the `localnet` network, which
does **not** have VLAN encapsulated traffic, while the other two will be
connected to the `externalvlan` network, which encapsulates traffic with the
VLAN tag 80.

One of the VMs connected to each of the aforementioned networks will be
scheduled in each of the nodes - i.e. one `tenantblue` connected VM will be
scheduled in `multi-homing-worker-0`, and the other in `multi-homing-worker-1`.
The same is true for the `tenantred` connected VMs. A `nodeSelector` is used to
implement this.

```yaml
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-sharing-br-no-vlan-1
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-0
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: no-vlan
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: no-vlan
        multus:
          networkName: novlan
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-sharing-br-no-vlan-2
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-1
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: no-vlan
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: no-vlan
        multus:
          networkName: novlan
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-sharing-br-with-vlan-1
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-0
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: external-vlan
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: external-vlan
        multus:
          networkName: underlay
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-sharing-br-with-vlan-2
spec:
  running: true
  template:
    spec:
      nodeSelector:
        kubernetes.io/hostname: multi-homing-worker-1
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: external-vlan
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: external-vlan
        multus:
          networkName: underlay
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:devel
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
```

Now you can ping between the VMs in each network, plus access services on the
underlay, as long as they are reachable via the GW IP.

The VLANs and subnets must match what was configured in the previous sections:
| Network Name | VLAN | Subnet           |
| ------------ | ---- | --------------   |
| externalvlan | 80   | 192.168.200.0/24 |
| novlan       | 0    | 192.168.122.0/24 |

Also remember the gateway IP for those subnets must be excluded.

## Conclusion
In this blog post we've seen how to configure secondary networks using an SDN
network fabric (implemented by OVN-Kubernetes) connected to the physical
underlay of the Kubernetes cluster. To simplify the underlay configuration, we
have used Kubernetes-NMState, which provides a policy layer that will apply a
desired configuration across the nodes which compose your cluster.

Two types of comprehensive examples were shown: one for clusters with a vacant
network interface, and another example for when the cluster features a single
network interface that must be shared between the cluster default network, and
the secondary networks.

