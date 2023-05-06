# Effortless Deployment of Rancher RKE2, Rancher Manager, Longhorn, and Neuvector

![rancher-long-banner](/images/rgs-banner-rounded.png)

### Table of Contents
* [About Me](#about-me)
* [Introduction](#introduction)
* [Infrastructure](#infrastructure)
* [RKE2 Configuration](#rke2-configuration)
* [Rancher Configuration](#rancher-configuration)
* [Longhorn Configuration](#longhorn-configuration)
* [NeuVector Configuration](#neuvector-configuration)
* [Final Thoughts](#final-thoughts)


## About Me

A little bit about me, my history, and what I've done in the industry. 
- DOD/IC Contractor
- U.S. Military Veteran
- Open-Source Contributor
- Built and Exited a Digital Firm
- Active Volunteer Firefighter/EMT


## Introduction

### Welcome to the Effortless Rancher Installation Guide

In this deployment guide, we will be installing the entire Rancher Stack to include the following products:

- RKE2 (Kubernetes Engine) - [click learn more](https://www.rancher.com/products/rke)
- Rancher MCM (Cluster Management) - [click to learn more](https://www.rancher.com/products/rancher)
- Longhorn (Storage) - [click to learn more](https://www.rancher.com/products/longhorn)
- Neuvector (Security) - [click to learn more](https://ranchergovernment.com/neuvector)
- and various minor tools/dependencies (more info below)


### Prerequisites

* Three (3) Internet Connected Linux Servers
  * Here's a list of our [supported operating systems](https://docs.rke2.io/install/requirements#operating-systems)
* Terminal Utility (Terminal, VSCode, Termius etc...)


## Infrastructure

For this deployment and installation, we need three linux servers to be able to get everything up and running. I will be using three Rocky Linux 9.1 machines, located on the same network. Any linux distribution should work perfectly fine, as long as their is network connectivity between them.

In order to configure these servers for Rancher, we will need these servers to be internet connected and accessible from your local device via `ssh`. If you would like to see my guide for an airgapped/offline installation, please check out my guide [here](https://github.com/zackbradys/rancher-offline).

| hostname | ip address | cores | memory | storage |
| :----: | :----: | :----: | :----: | :----: |
| `rke2-cp-01` | `10.0.0.15` | `4 cores` | `8 gbs` | `128 gbs` | 
| `rke2-wk-01` | `10.0.0.16` | `4 cores` | `8 gbs` | `128 gbs` |
| `rke2-wk-02` | `10.0.0.17` | `4 cores` | `8 gbs` | `128 gbs` |

Let's run the following commands on each of the nodes to ensure they have the neccessary packages. 

*Note: These commands are specific to Rocky Linux 9.1*

```bash
### Install Packages
yum install -y zip zstd tree jq cryptsetup

yum --setopt=tsflags=noscripts install -y nfs-utils

yum --setopt=tsflags=noscripts install -y iscsi-initiator-utils && echo "InitiatorName=$(/sbin/iscsi-iname)" > /etc/iscsi/initiatorname.iscsi && systemctl -q enable iscsid && systemctl start iscsid

yum update -y && yum clean all
```


## RKE2 Configuration

In order to configure and install Rancher RKE2, you need to have Control/Server nodes and Worker/Agent nodes. We will start by setting up the Control/Server node and then setting up the Worker/Agent nodes. There are many ways to accomplish this and this guide is meant for an effortless and simple installation, please review the [rke2 docs](https://docs.rke2.io) for more information.

### RKE2 Control Node

Let's start by configuring the RKE2 Control/Server Node, by adding the configuration file. Since we are completing a effortless installation, we are only adding the RKE2 Token configuration option. I'm using `ssh` with `root` to access the `rke2-cp-01` server.

If you would like to see more ways to configure the RKE2 Control/Server, please check out the [rke2 server docs](https://docs.rke2.io/reference/server_config).

```bash
### rke2-cp-01
### Create the RKE2 Directory
mkdir -p /etc/rancher/rke2/ 

### Create the RKE2 Configuration File
cat << EOF >> /etc/rancher/rke2/config.yaml
token: rke2SecurePassword
EOF
```

Now that the configuration file is completed, let's download and start the RKE2 Control/Server Node:

```bash
### rke2-cp-01
### Download the RKE2 Control/Server Binary
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.24.13 INSTALL_RKE2_TYPE=server sh - 

### Start the RKE2 Control/Server Service
systemctl enable rke2-server.service && systemctl start rke2-server.service
```

Let's verify that the RKE2 Control/Server is running using `systemctl status rke2-server`. It should look like this:

**IMAGE**

Now that we see that the RKE2 Control/Server is running, let's verify it using `kubectl`. 

```bash
### rke2-cp-01
### Symlink kubectl and containerd
sudo ln -s /var/lib/rancher/rke2/data/v1*/bin/kubectl /usr/bin/kubectl
sudo ln -s /var/run/k3s/containerd/containerd.sock /var/run/containerd/containerd.sock

### Update your paths in .bashrc
cat << EOF >> ~/.bashrc
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml 
export PATH=$PATH;/var/lib/rancher/rke2/bin;/usr/local/bin/
EOF
source ~/.bashrc

### Verify status with kubectl
kubectl get nodes -o wide
```

It should look like this:

**IMAGE**


### RKE2 Worker Nodes

Again, let's start by configuring the RKE2 Worker/Agent Nodes, by adding the configuration file. Since we are completing a effortless installation, we are only adding the RKE2 Server and RKE2 Token configuration options. I'm using `ssh` with `root` to access the `rke2-wk-01` and `rke2-wk-02` servers.

*Note: You need to complete each of these steps on each worker/agent server.*

If you woud like to see more ways to configure the RKE2 Worker/Agent, please check out the [rke2 agent docs](https://docs.rke2.io/reference/linux_agent_config).

```bash
### ke2-wk-01 and rke2-wk-02
### Create the RKE2 Directory
mkdir -p /etc/rancher/rke2/ 

### Create the RKE2 Configuration File
cat << EOF >> /etc/rancher/rke2/config.yaml
server: https://10.0.0.15:9345
token: rke2SecurePassword
EOF
```

Now that the configuration file is completed, let's download and start the RKE2 Worker/Agent Nodes:

```bash
### rke2-wk-01 and rke2-wk-02
### Download the RKE2 Worker/Agent Binary
curl -sfL https://get.rke2.io | INSTALL_RKE2_CHANNEL=v1.24.13 INSTALL_RKE2_TYPE=agent sh -

### Start the RKE2 Worker/Agent Service
systemctl enable rke2-agent.service && systemctl start rke2-agent.service
```

Let's head back to the `rke2-cp-01` server and verify the worker/agent nodes sucessfully joined the cluster.

```bash
### Verify status with kubectl
kubectl get nodes -o wide
```

It should look like this:

**IMAGE**

Congraulations!! In a few minutes, you now have a Rancher RKE2 Kubernetes Cluster up and running! If you are already familiar with Kubernetes or RKE2, feel free to explore the cluster using `kubectl`. We are going to move onto installing the [Rancher Multi Cluster Manager](https://www.rancher.com/products/rancher), [Rancher Longhorn](https://www.rancher.com/products/longhorn), and [Rancher NeuVector](https://ranchergovernment.com/neuvector).


## Rancher Configuration



## Longhorn Configuration



## NeuVector Configuration



## Final Thoughts


