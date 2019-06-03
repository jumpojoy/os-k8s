Prerequesites
=============

Working kubernetes cluster with multiple computes where node labeling is done according to theirs roles.
For openstack we will require the following labels:
 * openstack-control-plane=enabled - for k8s computes that will host openstack control plane containers
 * openstack-compute-node=enabled - for k8s computes that will host openstack compute nodes
 * openvswitch=enabled - for k8s computes that will host openstack network gateway services and compute nodes

For ceph we will require the following labels:
 * role=ceph-osd-node - for k8s computes that will host ceph osd's

Example of node labeling:

root@ctl01:/home/vsaienko# kubectl get nodes --show-labels
NAME    STATUS   ROLES    AGE     VERSION                    LABELS
cmp1    Ready    node     3h37m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,extraRuntime=virtlet,kubernetes.io/hostname=cmp1,node-role.kubernetes.io/node=true,openstack-control-plane=enabled,role=ceph-osd-node
cmp2    Ready    node     3h37m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,extraRuntime=virtlet,kubernetes.io/hostname=cmp2,node-role.kubernetes.io/node=true,openstack-control-plane=enabled,role=ceph-osd-node
cmp3    Ready    node     3h36m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,extraRuntime=virtlet,kubernetes.io/hostname=cmp3,node-role.kubernetes.io/node=true,openstack-compute-node=enabled,openvswitch=enabled,role=ceph-osd-node
ctl01   Ready    master   3h42m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ctl01,node-role.kubernetes.io/master=true
ctl02   Ready    master   3h41m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ctl02,node-role.kubernetes.io/master=true
ctl03   Ready    master   3h41m   v1.13.5-3+98374c02d2d8c1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=ctl03,node-role.kubernetes.io/master=true

Deploy helmbundle controller
============================

The controller manages releases of helm charts.

kubectl apply -f crds/helmbundle/helmcontroller.yaml

Deploy rook controller
======================

kubectl apply -f crds/helmbundle/ceph/rook.yaml

Deploy ceph cluster
===================

kubectl apply -f crds/ceph/cluster.yaml
kubectl apply -f crds/ceph/storageclass.yaml

Share ceph metadata with openstack
==================================

#Import secret from ceph-rook namespace
kubectl get secret rook-ceph-admin-keyring -n rook-ceph --export -o yaml | sed -e 's/keyring:/key:/' | kubectl apply -n default -f-

#Import ceph.conf configmap as well

kubectl cp rook-ceph/$(kubectl -n rook-ceph get pod -l "app=rook-ceph-operator" -o jsonpath='{.items[0].metadata.name}'):/etc/ceph/ceph.conf /tmp/ceph.conf && sed -i 's/[a-z]1\+://g' /tmp/ceph.conf; sed -i '/^\[client.admin\]/d' /tmp/ceph.conf; sed -i '/^keyring =/d' /tmp/ceph.conf; sed -i '/^$/d' /tmp/ceph.conf;

kubectl create configmap rook-ceph-config -n default --from-file=/tmp/ceph.conf

Deploy infra components
=======================

kubectl apply -f crds/helmbundle/openstack/ceph/infra.yaml

Deploy storage independeng  openstack services
==============================================


kubectl apply -f crds/helmbundle/openstack/common/openstack.yaml

Deploy rest of core openstack
=============================

kubectl apply -f crds/helmbundle/openstack/ceph/glance.yaml
kubectl apply -f crds/helmbundle/openstack/ceph/cinder.yaml
kubectl apply -f  crds/helmbundle/openstack/common/openstack-network.yaml
kubectl apply -f  crds/helmbundle/openstack/common/openstack-compute.yaml


Validate OpenStack
==================

mkdir /etc/openstack
tee /etc/openstack/clouds.yaml << EOF
clouds:
  openstack_helm:
    region_name: RegionOne
    identity_api_version: 3
    auth:
      username: 'admin'
      password: 'password'
      project_name: 'admin'
      project_domain_name: 'default'
      user_domain_name: 'default'
      auth_url: 'http://keystone.default.svc.cluster.local/v3'
EOF

export OS_CLOUD=openstack_helm

apt-get install virtualenv build-essential python-dev
virtualenv osclient
. osclient/bin/activate
pip install python-openstackclient

wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openstack image create cirros-0.4.0-x86_64-disk --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare
openstack network create demoNetwork
openstack subnet create demoSubnet --network demoNetwork --subnet-range 10.11.12.0/24
openstack server create --image cirros-0.4.0-x86_64-disk --flavor m1.tiny --nic net-id=demoNetwork DemoVM
