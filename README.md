# Ansible Playbooks To setup cross-datacenters Kubernetes HA (multi-master) on Ubuntu 24.04 LTS

This repository provides Ansible Playbooks to setup Kubernetes HA on Ubuntu 24.04 LTS. The playbooks are mainly inspired by Kubeadm documentation and other ansible tentatives on github. The playbooks could be used separately or as one playbook for a fully fledged HA cluster.


# Prerequisites:

Ubuntu 24.04 LTS

On your manager machine install Python and pip:
```
sudo apt update
sudo apt install python3 python3-pip
```


Install ansible on your ansible manager machine.

* You can do:
```
pip3 install ansible
```

* Setup ssh access from master to workers.
```ssh-copy-id -i ~/.ssh/id_rsa.pub <user@host>```

# Environment preparation:

* Clone the repo:
 In the machine that you want to use as ansible manager (can be your laptop or any other machine that has ssh access to the target machines):
 ```
 git clone git@github.com:IBM/ansible-kubernetes-ha-cluster.git
 cd ansible-k8s-ha
 ```

* Create inventory/mycluster
and declare your machines such as:
```
myhostname.domain.com ansible_usehost=<ip>
```

Also make sure to update the `vars` section:
- choose the desired versions for kubernetes and CRI-O
- setup the pod network cidr (default setup is for flannel)
- Setup the eviction hard properties for workers and masters
- Choose whether you want to use a private setup or a public one, with a private setup, you need to provide the credentials to get the apt packages and container images.

There are different groups being defined and used, you can reuse mycluster file defined in inventory folder:
```
[dc1-k8s-masters] # these are all the masters of datacenter 1 (DC1)
[dc1-k8s-workers-vm] # these are all the VM worker nodes of DC1
[dc1-k8s-workers-bm] # these are all baremetal worker nodes of DC1
```
We can have as many data centers as we need. For each data center, define the masters and workers and add them to `[k8s-masters:children]`, `[k8s-workers:children]`, and `[k8s-nodes:children]`.

You can check that you can ping all the machines:

```
ansible -m ping all -i inventory/mycluster
```
# Install a highly available kubernetes using kubeadm

You can now run k8s-all playbook to get your cluster setup.
You can also run the different playbooks separately for different purposes (setting up CRI-O, etcd, keepalived, kubeadm ...).

```
ansible-playbook -i inventory/mycluster  playbooks/k8s-all.yaml
```

# What k8s-all.yaml includes:

- Adding the required apt repository (private or public)
- Installing chrony for time synchronization
- Installing CRI-O container runtime
- Installing kubeadm, kubelet and kubectl
- Setting up ufw firewall
- Generating etcd certificates and installing ha etcd cluster on all the master nodes
- Installing haproxy
- Installing keepalived and setting up vip management (this optional, use only when you have a vip)
- Setting up kubernetes masters
- Adding the nodes to the cluster
- Reconfiguring the nodes and components to communicate through haproxy
- Encrypting kubernetes secrets at rest
- Running a smoke test

# Restarting the install:

If you need to restart the process using kubeadm reset, please use the cleanup-all-vms playbook that deletes the state from all vms. Some of the commands might fail but you can ignore that.

# Encrypting Secrets at rest (already in the k8s-all playbook):

If you want to add an extra layer of securing your secrets by encrypting them at rest you can use the "encrypting-secrets.yaml playbook". You can add it to the k8s-all.yaml or use it separately.

Before using it, update the inventory file to change the encryption key variable "encoded_secret".
To generate a new encryption key you can do the following:

```
head -c 32 /dev/urandom | base64
```
Copy the output and save it in the inventory variable.

After that run the playbook:

```
ansible-playbook -i inventory/mycluster  playbooks/encrypting-secrets.yaml
```

# Supported Versions:

- Kubernetes 1.35.x
- CRI-O 1.35
- Ubuntu 24.04 LTS

# Contribution:

In order to contribute please feel free to create github issues and PRs.
