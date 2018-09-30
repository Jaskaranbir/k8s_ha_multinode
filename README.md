## High-Availability Multi-Node Kubernetes Cluster
---

A **completely Dockerized** multi-node Kubernetes highly-available cluster provisioned using Vagrant/Ansible, based on Kubernetes version **1.12** (still not enough fancy words for a day :smiley:)

**Note**: This is not a production-ready setup. Instead, this is intended to be a base/idea for one (if looking for custom setups, otherwise [Kubeadm][0] does job pretty well).

### How Stuff Works

#### Kubernetes

* The setup uses multi-master and multi-worker setup (and multi-etcd, of course).

* On the master-node side, everything is ordinary, as you would expect from any regular Kubernetes master.

* On the worker-node side, the master-nodes are loadbalanced using HAProxy. So the Kubelet connects to HAProxy's address instead of a specific master.

* Yes, HAProxy runs on each of the worker-nodes instead of master. This is because if the master goes down, it also takes down loadbalancer with it (not an ideal scenario).

* CNI: [Weave Net][1]

* DNS: [Core DNS][2]

#### Vagrant

* Vagrant is simply a convenient way of automatically spinning up a cluster. You can easily configure the instances in `Vagrantfile`.

* Uses Virtualbox.

* Default instance-count:
```
ETCD: 1
Kube-Master: 1
Kube-Worker: 2
```

* The setup is based on a custom packed **CoreOS** based Vagrant-image. Image-source: [Jaskaranbir/packer_coreos-ansible-python][3]

* Just run `vagarnt up`, and it will automatically run install/run Ansible and setup a local Kubernetes cluster.

TODO: Improve cluster-security.
Suggestions welcomed.

### Ansible Notes

* When adding/removing instances, be sure to also update the Ansible [inventory][4].

* Ansible copies its templates for manifests/configs to `/etc/kubernetes`, which will contain all Kubernetes resources, including certificates.

  [0]: https://kubernetes.io/docs/setup/independent/install-kubeadm/
  [1]: https://www.weave.works/oss/net/
  [2]: https://coredns.io/
  [3]: https://github.com/Jaskaranbir/packer_coreos-ansible-python
  [4]: https://github.com/Jaskaranbir/k8s_ha_multinode/blob/master/inventory/hosts.yml
