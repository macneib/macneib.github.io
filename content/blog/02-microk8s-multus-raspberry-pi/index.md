---
title: Microk8s with Calico and Multus on a Raspberry pi cluster
date: "2021-02-24T22:12:03.284Z"
description: "Notes from setting up microk8s with calico and multus on a raspberry pi cluster"
---

### Introduction

This ultimately failed because multus for microk8s was disabled for ARM64 targets.
I will watch [this issue](https://github.com/ubuntu/microk8s/pull/1872) and try again this PR has been accepted.

Will try with k3sup as that has flannel-cni installed by default and people recomend that as a good starting place.


### Step by step instructions

change the password in each pi board

```
ssh ubuntu@192.168.1.16
exit

ssh ubuntu@192.168.1.17
exit

ssh ubuntu@192.168.1.18
exit

ssh ubuntu@192.168.1.19
exit
```

Add nodes in dev machines `/etc/hosts` files

```
sudo vim /etc/hosts
```

Paste the following snippet

```
192.168.1.16    master01 master01.microk8s
192.168.1.17    worker01 worker01.microk8s
192.168.1.18    worker02 worker02.microk8s
192.168.1.19    worker03 worker03.microk8s
```

install ansible on control machine
```
apt install ansible
```

```
sudo vim /etc/ansible/hosts
```

Paste the following snippet

```
[masters]
master01  ansible_connection=ssh   var_hostname=master01 ansible_ssh_private_key_file=/home/macneib/.ssh/id_ed25519 ansible_ssh_user=ubuntu
#master02  ansible_connection=ssh   var_hostname=master02
#master03  ansible_connection=ssh   var_hostname=master03

[workers]
worker01  ansible_connection=ssh  var_hostname=worker01 ansible_ssh_private_key_file=/home/macneib/.ssh/id_ed25519 ansible_ssh_user=ubuntu
worker02  ansible_connection=ssh  var_hostname=worker02 ansible_ssh_private_key_file=/home/macneib/.ssh/id_ed25519 ansible_ssh_user=ubuntu
worker03  ansible_connection=ssh  var_hostname=worker03 ansible_ssh_private_key_file=/home/macneib/.ssh/id_ed25519 ansible_ssh_user=ubuntu
#worker04  ansible_connection=ssh  var_hostname=worker04
#worker05  ansible_connection=ssh  var_hostname=worker05
#worker06  ansible_connection=ssh  var_hostname=worker06

[microk8s:children]
masters
workers
```

Generate an ssh key

```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Copy ssh key over to remote machines

```
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.16
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.17
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.18
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.19
```


Check if Ansible is working fine and can connect to all nodes

```
ansible microk8s -m ping
```

```
ansible microk8s -b -m shell -a "hostnamectl set-hostname {{ var_hostname }}"
ansible cube -b -m shell -a "hostnamectl status | grep hostname"
```

Update the OS

```
ansible microk8s -m apt -a "upgrade=yes update_cache=yes" --become
```

Enable c-groups so the kubelet will work out of the box
```
ansible microk8s -b -m shell -a "sed -i '$ s/$/ cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1/' /boot/firmware/cmdline.txt"
```
Quick restart to get all the machines updated

```
ansible microk8s -b -m shell -a "shutdown -r now"
```

Install the MicroK8s snap and components
```
ansible microk8s -b -m shell -a "sudo snap install microk8s --classic"
ansible microk8s -b -m shell -a "sudo microk8s.enable dns storage"
sudo reboot now
```

Wait 1 minute then check status of masters

```
ansible masters -b -m shell -a "sudo microk8s.kubectl get nodes"
```

Add workers to master
```
ansible masters -b -m shell -a "sudo microk8s.add-node"
ansible worker01 -b -m shell -a "microk8s join 192.168.1.16:25000/<token>"

ansible masters -b -m shell -a "sudo microk8s.add-node"
ansible worker02 -b -m shell -a "microk8s join 192.168.1.16:25000/<token>"

ansible masters -b -m shell -a "sudo microk8s.add-node"
ansible worker03 -b -m shell -a "microk8s join 192.168.1.16:25000/<token>"
```

```
ansible masters -b -m shell -a "sudo microk8s enable multus"
```

```
[WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo
master01 | FAILED | rc=1 >>
Nothing to do for `multus`.non-zero return code
```

And this is the point where I realized that multus is not enabled on microk8s
