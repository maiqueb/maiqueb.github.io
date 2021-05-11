---
title: "KubeVirt: IPv6 configuration plus ssh public key injection"
date: 2021-04-22
categories:
  - blog
tags:
  - ipv6
  - cloud-init
---

In this short post, I will show how to configure IPv6 for a VM running in
KubeVirt, and inject an ssh public key into it, via cloud-init.

The first question I should answer, is "why?"; shouldn't the reader simply
look at the
[documentation](https://kubevirt.io/user-guide/virtual_machines/accessing_virtual_machines/#static-ssh-key-injection-via-cloud-init)
for how to inject a public ssh key into the KubeVirt VM, extend that yaml
specification with the network configuration found in the
[KubeVirt examples](https://github.com/kubevirt/kubevirt/blob/ccfdfd80a12a18963e23cd131582134e3bd9b974/examples/vmi-masquerade.yaml#L40)
and be done with it?

The short answer is: "that will not work". The reason is simple; cloud-init
[only supports](https://cloudinit.readthedocs.io/en/latest/topics/network-config.html#network-configuration-sources)
the netplan format for the `NoCloud` datasource. Furthermore, the `ConfigDrive`
datasource implementation in KubeVirt is limited to storing a
`network_data.json` file, which would leave us with a nasty looking Openstack
metadata format looking network configuration.

While that would work, it would lead to more questions than answers - yes; we
would need to comply with the Openstack metadata network_data schema, which,
quite simply, does not make sense outside the openstack world.

As such, the simplest (I like simple) way out, is to inject an OS specific
script to configure the interface with IPv6 via the `user-data` section.

Please refer to the following example to check how to do it for Fedora
distributions:
```bash
# create the Kubernetes secret with the public key
kubectl create secret generic my-pub-key \
    --from-file=ssh-publickey=<path to pub key>

cat << END > ipv6-ssh-key-injection-vm.yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-with-ipv6-config
  name: vm-with-ipv6-config
spec:
  dataVolumeTemplates:
  - metadata:
      name: fedora-dv
    spec:
      pvc:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
        storageClassName: local
      source:
        registry:
          url: docker://quay.io/kubevirt/fedora-cloud-container-disk-demo
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-with-ipv6-config
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: disk0
          - disk:
              bus: virtio
            name: disk1
          interfaces:
          - masquerade: {}
            name: default
        machine:
          type: ""
        resources:
          requests:
            cpu: 1000m
            memory: 1G
      terminationGracePeriodSeconds: 0
      networks:
        - name: default
          pod: {}
      accessCredentials:
      - sshPublicKey:
          source:
            secret:
              secretName: my-pub-key
          propagationMethod:
            configDrive: {}
      volumes:
      - dataVolume:
          name: fedora-dv
        name: disk0
      - cloudInitConfigDrive:
          userData: |-
            #!/bin/bash
            echo "fedora" |passwd fedora --stdin
            nmcli conn add type ethernet \
                ifname eth0 \
                con-name eth0 \
                ipv4.method auto \
                ipv6.method manual \
                ipv6.addresses fd10:0:2::2/120 \
                ipv6.gateway fd10:0:2::1 && \
              nmcli conn up eth0
        name: disk1
END

# create the VM
kubectl create -f ipv6-ssh-key-injection-vm.yaml

# start the VM
virtctl start vm-with-ipv6-config
```
