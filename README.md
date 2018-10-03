# Setup

This project provisions and deploys a local Kubernetes cluster.

Prerequisites:
- VirtualBox 5.1.38
- Vagrant 2.1.5
  - Also [vagrant-hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager) 1.8.9, installed with `vagrant plugin install vagrant-hostmanager`
- Ansible 2.6.4
- 8GB of free memory

It will probably work with the same major version of any of the above (e.g. any VirtualBox 5 version),
but those are the versions it was tested with.

Similarly, it should work with any version of the xenial vagrant box, but it was tested with 20180917.0.0.

It assumes a single master and has not been tested with more.

## Build flannel if using ipsec

If you wish to use the flannel CNI plugin with its ipsec backend, you will have to build it locally
```
git clone https://github.com/coreos/flannel
cd flannel
make image REGISTRY=master:5000/coreos/flannel TAG=latest
```

The ansible playbooks will take care of getting your image onto the virtual cluster.

## Create and provision VMs

All configuration is done at the top of `Vagrantfile`.  Then run:

`vagrant up`

### Re-provisioning

If something went wrong with ansible or you've made playbook changes and you want to rerun it, either do

`vagrant provision --provision-with ansible`

or

`cd ansible && ansible-playbook -i ../.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory site.yml`

If you've made changes to Vagrantfile you should provision via the vagrant command, since it may need to update the dynamic ansible inventory.  Otherwise it may be more convenient to invoke ansible-playbook directly so you can provide extra arguments without having to edit Vagrantfile.

Useful ansible arguments:
- `-v` increases verbosity, use more `v`s for more verbosity
- `-t` taglist to limit scope of actions, tags include `prepare`, `master`, `nodes`, and `postkube`
- `-l` hostlist to limit to subset of hosts

### Provisioning implementation details

The Vagrantfile contains a bit of bootstraping to pave the way for invoking ansible.  Then vagrant invokes
`ansible-playbook site.yml`.  

site.yml is logically divided into these steps (which are also roles and tags):
- `prepare` does basic OS setup and package installation for all nodes
- `master` does master-specific setup, including starting kubernetes
- `nodes` finishes installing kubernetes on all nodes
- `postkube` assumes a working cluster and does post-cluster-creation tasks on the master

Since an ambition of this project is to facilitate playing with different CNI options, setting up CNI
is left until `postkube`.

## Interacting with the cluster via ssh

The simplest option is to work directly on the cluster, so that you don't have to tweak your host at all:

`vagrant ssh master`
`vagrant ssh node1 -- kubectl get pods`

You can also ssh/scp directly if need be:

`vagrant ssh-config > ssh-config`
`ssh -F ssh-config master date`
`scp -F ssh-config node1:/tmp/file /tmp`

## Interacting with the cluster via kubernetes API

The ssh method requires setting up ssh port forwards if you want to access
any services from the cluster on your host though, e.g. to see the kubernetes
dashboard.  You can instead call the kubernetes API server directly if you:
- install kubectl locally
- set env var (from your git checkout root)
```
export KUBECONFIG=`pwd`/ansible/tmp/kubeconfig.local
```
- edit /etc/hosts to add `kubernetes` as an alias for localhost, e.g.
```
  127.0.0.1	localhost kubernetes
```

Then you can simply run

`kubectl get pods`

## Interacting with the cluster via Prometheus

To see the Prometheus web interface reporting on network connectivity, go to localhost:31000
(or whatever you set host_prom_port to in Vagrantfile).