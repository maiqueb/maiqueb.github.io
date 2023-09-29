---
title: "SDN for all the networks! Micro-segmentation for virt workloads"
date: 2023-11-23
categories:
  - blog
tags:
  - SDN
  - security
  - networking
  - virtualization
---

## Background
Virtualization workloads commonly have to access services on the physical
network. Those same users also already have templated images available, with
configured IP addresses. Thus, relying on the cluster default network is not
an option - only secondary networks are.

Let's take this a step further; it is good practice to use micro-segmentation
to mitigate supply chain attacks - i.e. define using policy exactly what
services can a workload access / be accessed from. Thus, even if a component is
compromised, it wouldn't be able to contact the outside world.

The combination of these two features is quite common, and useful. And up to
now, impossible to have in Openshift: there simply wasn't support for Kubernetes
`NetworkPolicy` on secondary networks, least of all on the SDN solution offered
by Openshift.

This blog post will explain how to configure a secondary L2 overlay connected
to a physical network, and how to specify exactly what the workloads attached to
these physical networks can do - i.e. which services can they access / which
services get to access them.

## Requirements
- an Openshift 4.14 cluster
- kubernetes-nmstate >= 2.2.15
- CNO configuration:
  - enabled multi-network
  - enabled multi-net policies
  - OVN-Kubernetes plugin
- a postgreSQL DB installed, available in the physical network

## Scenario
In this scenario we will deploy workloads (both pods and VMs) requiring access
to a relational DB reachable via the physical network (i.e. deployed outside
Kubernetes).

For that we will deploy a secondary network.

### Overlay network perspective
The VMs will be connected to two different networks: the cluster's default
network, owned and managed by Kubernetes, granting the VMs access to the
internet, and an additional secondary network, named `tenantblue`, implemented
by OVN-Kubernetes, and connected to the physical network, through which it will
access the database deployed outside Kubernetes.

Both VMs (the the DB) will be available on the same subnet (192.168.200.0/24).
The diagram below depicts the scenario explained above.

![overlay-network-view](../assets/ovn-k-secondary-nets-localnet-overlay-view.png 'Logical network view')

### Physical network perspective
![underlay-network-view](../assets/ovn-k-secondary-nets-localnet-underlay-view.png 'Physical network view')

## Setting up the database
TODO

## Creating an overlay network connected to a physical network
The first thing we need to do is configure the underlay for our "new" network;
we will use an NMState node network configuration policy for that.

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-share-same-gw-bridge
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''
  desiredState:
    ovn:
      bridge-mappings:
      - localnet: tenantblue
        bridge: br-ex
```

The policy above will be applied only on worker nodes - notice the node selector
 used - and will configure a mapping in the OVS system to connect the traffic
from the `tenantblue` network to the `br-ex` OVS bridge, which is already
configured - and managed - by OVN-Kubernetes. Do not touch any of its settings -
all you can do is point more networks to it.

Now we need to provision the logical networks in Openshift. For that, please use
the following manifests:
```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: tenantblue
spec:
    config: '{
        "cniVersion": "0.3.1",
        "name": "tenantblue",
        "netAttachDefName": "default/tenantblue",
        "topology": "localnet",
        "type": "ovn-k8s-cni-overlay",
        "logFile": "/var/log/ovn-kubernetes/ovn-k8s-cni-overlay.log",
        "logLevel": "5",
        "logfile-maxsize": 100,
        "logfile-maxbackups": 5,
        "logfile-maxage": 5
    }'
```

**NOTE:** the `spec.config.name` of the NAD - i.e. `tenantblue` - must match the
`localnet` attribute of the ovn bridge mapping provisioned above.

Now we just need to provision the workloads:
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: podclient
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      {
        "name": "tenantblue",
        "ips": [ "192.168.200.10/24" ]
      }
    ]'
spec:
  containers:
  - name: pod2
    image: k8s.gcr.io/e2e-test-images/agnhost:2.26
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    securityContext:
      runAsUser: 1000
      privileged: false
      seccompProfile:
        type: RuntimeDefault
      capabilities:
        drop: ["ALL"]
      runAsNonRoot: true
      allowPrivilegeEscalation: false
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-server
spec:
  running: true
  template:
    spec:
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
          - name: default
            masquerade: {}
          - name: underlay
            bridge: {}
        machine:
          type: ""
        resources:
          requests:
            memory: 1024M
      networks:
      - name: default
        pod: {}
      - name: underlay
        multus:
          networkName: tenantblue
      terminationGracePeriodSeconds: 0
      volumes:
        - name: containerdisk
          containerDisk:
            image: quay.io/containerdisks/fedora:38
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              version: 2
              ethernets:
                eth0:
                  dhcp4: true
                eth1:
                  gateway4: 192.168.200.1
                  addresses: [ 192.168.200.20/24 ]
            userData: |-
              #cloud-config
              password: fedora
              chpasswd: { expire: False }
```

## Restrict traffic on the secondary network
TODO

## Conclusions
In this blog post we have seen how the user can deploy a VM workload attached to
a secondary network that requires access to a service deployed outside
Kubernetes - in this scenario, an SQL database.

We have also seen how to use network policies to restrict access in this
secondary network, only allowing access to the database itself.

