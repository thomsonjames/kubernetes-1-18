# Kubernetes 1.18 and Container Ingress Controller clusterIP Quick Start Guide

This page is created to document K8S 1.18 with integration of CIS and BIG-IP. Please contact me at m.dittmer@f5.com if you have any questions

# Note

Environment parameters

* K8S 1.18 - one master and two worker nodes
* CIS 1.14
* AS3: 3.17.1
* BIG-IP 14.1.2 (standalone deployment)

# Kubernetes 1.18 Install

K8S is installed on RHEL 7.5 on ESXi

* ks8-1-18-master  
* ks8-1-18-node1
* ks8-1-18-node2

## Prerequisite

Since CIS is using the AS3 declarative API we need the AS3 extension installed on BIG-IP. Follow the link to install AS3
 
* Install AS3 on BIG-IP
https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html

##### Initial Setup for BIG-IP

BIG-IP is connecting to the K8S cluster using cluster mode and therefore BIG-IP needs to be part of container network infrastructure (CNI). Choices are BGP or VXLAN. This quick start guide is created using VXLAN (Flannel). BIG-IP tmsh commands below creates the partition and VXLAN tunnel. CIS needs a partition on BIG-IP for FDB entries and ARP requests 

```
tmsh create auth partition k8s
tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type none
tmsh create net tunnels tunnel fl-vxlan key 1 profile fl-vxlan local-address 192.168.200.92
tmsh create net self 10.244.20.92 address 10.244.20.92/255.255.0.0 allow-service none vlan fl-vxlan
```

## Deploy flannel for Kubernetes

Add the BIG-IP device to the flannel overlay network. Find the VTEP MAC address

```
root@(bip-ip-ve2-pme)(cfg-sync Standalone)(Active)(/Common)(tmos)# show net tunnels tunnel fl-vxlan all-properties

-------------------------------------------------
Net::Tunnel: fl-vxlan
-------------------------------------------------
MAC Address                   **00:50:56:bb:70:8b**
Interface Name                           fl-vxlan

Incoming Discard Packets                        0
Incoming Error Packets                          0
Incoming Unknown Proto Packets                  0
Outgoing Discard Packets                        0
Outgoing Error Packets                         10
HC Incoming Octets                              0
HC Incoming Unicast Packets                     0
HC Incoming Multicast Packets                   0
HC Incoming Broadcast Packets                   0
HC Outgoing Octets                              0
HC Outgoing Unicast Packets                     0
HC Outgoing Multicast Packets                   0
HC Outgoing Broadcast Packets                   0
```

## Create a “dummy” Kubernetes Node for the BIG-IP device

Include all of the flannel Annotations. Define the backend-data and public-ip Annotations with data from the BIG-IP VXLAN:

```
apiVersion: v1
kind: Node
metadata:
  name: bigip1
  annotations:
    #Replace MAC with your BIGIP Flannel VXLAN Tunnel MAC
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:50:56:bb:70:8b"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    #Replace IP with Self-IP for your deployment
    flannel.alpha.coreos.com/public-ip: "192.168.200.92"
spec:
  #Replace Subnet with your BIGIP Flannel Subnet
  podCIDR: "10.244.20.0/24
```

## Create CIS Controller, BIG-IP credentials and RBAC Authentication

Configuration options available in the CIS controller using user-defined configmap
```
args: 
     - "--bigip-username=$(BIGIP_USERNAME)"
     - "--bigip-password=$(BIGIP_PASSWORD)"
     - "--bigip-url=192.168.200.92"
     - "--bigip-partition=k8s"
     - "--namespace=default"
     - "--pool-member-type=cluster"   ----- As per code it will process as clusterIP
     - "--flannel-name=fl-vxlan"
     - "--log-level=DEBUG"
     - "--insecure=true"
     - "--manage-ingress=false"
     - "--manage-routes=false"
     - "--agent=as3"
     - "--as3-validation=true"
```

## BIG-IP credentials and RBAC Authentication

```
#create kubernetes bigip container connecter, authentication and RBAC
kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=f5PME123
kubectl create serviceaccount k8s-bigip-ctlr -n kube-system
kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr
kubectl create -f f5-cluster-deployment.yaml
kubectl create -f f5-bigip-node.yaml
```