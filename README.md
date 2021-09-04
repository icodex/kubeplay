# kubeplay

[![Build kubeplay packages](https://github.com/k8sli/kubeplay/actions/workflows/build.yaml/badge.svg?branch=main)](https://github.com/k8sli/kubeplay/actions/workflows/build.yaml)

> English | [简体中文](./README_zh-CN.md)

## Introduction

[kubeplay](https://github.com/k8sli/kubeplay) is a tool for offline deployment of kuberneres clusters based on [kubespray](https://github.com/k8sli/kubespray).

### Feature

- All dependencies  included, installing offline with one single command
- amd64 and arm64 CPU architectures supported
- Validity of certificate generated by kubeadm extended to 10 years
- No docker dependency, seamless migrating  container runtime to containerd
- Ideal for toB privatized deployment scenarios as all rpm/deb packages (e.g. storage client) needed by bootstraping cluster can be installed offline
- Multi-cluster deployment supported, deploying a new kubernetes cluster with Job Pods within kubernetes cluster
- Offline installer built with GitHub Actions, no charge, 100% open source 100% free

### Version of addons

| addon        | version        | usage                           |
| ------------ | -------------- | ------------------------------- |
| kubernetes   | v1.21.4        | kubernetes                      |
| containerd   | v1.4.6         | container runtime               |
| etcd         | v3.4.13        | etcd service                    |
| crictl       | v1.21.0        | CRI CLI tool                    |
| pause        | 3.3            | pause container image           |
| cni-plugins  | v0.9.1         | CNI plugins                     |
| calico       | v3.18.5        | calico                          |
| autoscaler   | 1.8.3          | DNS auto scaling                |
| coredns      | v1.8.0         | cluster DNS service             |
| flannel      | v0.14.0        | flannel                         |
| nginx        | 1.19           | reverse proxy of APIserver      |
| canal        | calico/flannel | calico and flannel intergration |
| helm         | v3.6.3         | helm CLI tool                   |
| nerdctl      | 0.8.0          | containerd CLI tool             |
| nerdctl-full | 0.11.0         | containerd toolset              |
| registry     | v2.7.1         | container image registry        |
| skopeo       | v1.4.0         | image porting tool              |

### Supported Linux Distributions

| distribution | version     | arch        |
| ------------ | ----------- | ----------- |
| CentOS       | 7/8         | amd64/arm64 |
| Debian       | 9/10        | amd64/arm64 |
| Ubuntu       | 18.04/20.04 | amd64/arm64 |
| Fedora       | 33/34       | amd64/arm64 |


### compose

Running nginx and registry with [nerdctl compose](https://github.com/containerd/nerdctl) on deploy node where deployment tool would run, which provide offline resource download and image distribution services.

### kubespray

Using [kubespray](https://github.com/kubernetes-sigs/kubespray) which come from kubernetes community as a cluster deployment executor, needed resources  during deployment will fetched from compose node.

## How to deploy

### Download

You can download the package matching your platform from the releases page [k8sli/kubeplay/releases](https://github.com/k8sli/kubeplay/releases) on GitHub, then copy it to deploy node.

```bash
kubeplay-v0.1.0-alpha.3-centos-7.sha256sum.txt # checksum file
kubeplay-v0.1.0-alpha.3-centos-7-amd64.tar.gz  # for  CentOS 7 amd64
kubeplay-v0.1.0-alpha.3-centos-7-amd64.tar.gz  # for CentOS 7 arm64
```

### Configuration

```bash
$ tar -xpf kubeplay-x.y.z-xxx-xxx.tar.gz
$ cd kubeplay
$ cp config-sample.yaml config.yaml
$ vi config.yaml
```

The `config.yaml` configuration file is divided into the following sections：

- compose：config for nginx and registry on current deploy node
- kubespray：kubespray deployment config
- invenory：ssh config for nodes of kubernetes cluster
- default：default config values

#### compose

| parameter       | description                 | example             |
| --------------- | --------------------------- | ------------------- |
| internal_ip     | reachable IP of deploy node | 192.168.10.11       |
| nginx_http_port | nginx serve port            | 8080                |
| registry_domain | registry service domain     | kube.registry.local |

```yaml
compose:
  # Compose bootstrap node ip, default is local internal ip
  internal_ip: 172.20.0.25
  # Nginx http server bind port for download files and packages
  nginx_http_port: 8080
  # Registry domain for CRI runtime download images
  registry_domain: kube.registry.local
```

#### kubespray

| parameter                    | description                  | example        |
| ---------------------------- | ---------------------------- | -------------- |
| kube_version                 | kubernetes version           | v1.21.3        |
| external_apiserver_access_ip | external APIserver access IP | 192.168.10.100 |
| kube_network_plugin          | selected CNI plugin          | calico         |
| container_manager            | container runtime            | containerd     |
| etcd_deployment_type         | etcd deployment mode         | host           |

```yaml
kubespray:
  # Kubernetes version by default, only support v1.20.6
  kube_version: v1.21.3
  # For deploy HA cluster you must configure a external apiserver access ip
  external_apiserver_access_ip: 127.0.0.1
  # Set network plugin to calico with vxlan mode by default
  kube_network_plugin: calico
  #Container runtime, only support containerd if offline deploy
  container_manager: containerd
  # Now only support host if use containerd as CRI runtime
  etcd_deployment_type: host
  # Settings for etcd event server
  etcd_events_cluster_setup: true
  etcd_events_cluster_enabled: true
```

#### inventory

inventory is the ssh login configuration for nodes of kubernetes cluster , supporting yaml, json, and ini formats.

| parameter                    | description                                                  | example                                        |
| ---------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| ansible_port                 | ssh port using by ansible                                    | 22                                             |
| ansible_user                 | ssh user using by ansible                                    | root                                           |
| ansible_ssh_pass             | ssh password using by ansible                                | password                                       |
| ansible_ssh_private_key_file | ssh private key using by ansible if choose to login without password | only valid value is `/kubespray/config/id_rsa` |
| ansible_host                 | node IP                                                      | 172.20.0.21                                    |

- yaml format

```yaml
# Cluster nodes inventory info
inventory:
  all:
    vars:
      ansible_port: 22
      ansible_user: root
      ansible_ssh_pass: Password
      # ansible_ssh_private_key_file: /kubespray/config/id_rsa
    hosts:
      node1:
        ansible_host: 172.20.0.21
      node2:
        ansible_host: 172.20.0.22
      node3:
        ansible_host: 172.20.0.23
      node4:
        ansible_host: 172.20.0.24
    children:
      kube_control_plane:
        hosts:
          node1:
          node2:
          node3:
      kube_node:
        hosts:
          node1:
          node2:
          node3:
          node4:
      etcd:
        hosts:
          node1:
          node2:
          node3:
      k8s_cluster:
        children:
          kube_control_plane:
          kube_node:
      gpu:
        hosts: {}
      calico_rr:
        hosts: {}
```

- json format

```json
inventory: |
  {
    "all": {
      "vars": {
        "ansible_port": 22,
        "ansible_user": "root",
        "ansible_ssh_pass": "Password"
      },
      "hosts": {
        "node1": {
          "ansible_host": "172.20.0.21"
        },
        "node2": {
          "ansible_host": "172.20.0.22"
        },
        "node3": {
          "ansible_host": "172.20.0.23"
        },
        "node4": {
          "ansible_host": "172.20.0.24"
        }
      },
      "children": {
        "kube_control_plane": {
          "hosts": {
            "node1": null,
            "node2": null,
            "node3": null
          }
        },
        "kube_node": {
          "hosts": {
            "node1": null,
            "node2": null,
            "node3": null,
            "node4": null
          }
        },
        "etcd": {
          "hosts": {
            "node1": null,
            "node2": null,
            "node3": null
          }
        },
        "k8s_cluster": {
          "children": {
            "kube_control_plane": null,
            "kube_node": null
          }
        },
        "gpu": {
          "hosts": {}
        },
        "calico_rr": {
          "hosts": {}
        }
      }
    }
  }
```

- ini fomat

```ini
inventory: |
  [all:vars]
  ansible_port=22
  ansible_user=root
  ansible_ssh_pass=Password
  #ansible_ssh_private_key_file=/kubespray/config/id_rsa

  [all]
  kube-control-01 ansible_host=172.20.0.21
  kube-control-02 ansible_host=172.20.0.23
  kube-control-03 ansible_host=172.20.0.22
  kube-node-01 ansible_host=172.20.0.24

  [bastion]
  # bastion-01 ansible_host=x.x.x.x ansible_user=some_user

  [kube_control_plane]
  kube-control-01
  kube-control-02
  kube-control-03

  [etcd]
  kube-control-01
  kube-control-02
  kube-control-03


  [kube_node]
  kube-control-01
  kube-control-02
  kube-control-03
  kube-node-01

  [calico_rr]

  [k8s_cluster:children]
  kube_control_plane
  kube_node
  calico_rr
```

#### default value

The following default parameters are not recommended to be modified without special requirements, just leave them as default. Unmodified `ntp_server` value will be overrided by `internal_ip` from compose section; `registry_ip` and `offline_resources_url` are automatically generated based on the parameters in compose section thus not need to modify.

| parameter                 | description                                                  | example |
| ------------------------- | ------------------------------------------------------------ | :-----: |
| ntp_server                | ntp clock synchronization server domain or IP                |    -    |
| registry_ip               | registry IP                                                  |    -    |
| offline_resources_url     | URL address for downloading offline resources                |    -    |
| offline_resources_enabled | whether to deploy offline                                    |  true   |
| generate_domain_crt       | whether to generate self-signed certificate for mirror repository domain |  true   |
| image_repository          | repo or project of image registry                            | library |
| registry_https_port       | port of registry，PUSH operation disabled                    |   443   |
| registry_push_port        | port of registry for PUSH                                    |  5000   |
| download_container        | whether to pull images of all components under all nodes     |  false  |

```yaml
default:
  # NTP server ip address or domain, default is internal_ip
  ntp_server:
    - internal_ip
  # Registry ip address, default is internal_ip
  registry_ip: internal_ip
  # Offline resource url for download files, default is internal_ip:nginx_http_port
  offline_resources_url: internal_ip:nginx_http_port
  # Use nginx and registry provide all offline resources
  offline_resources_enabled: true
  # Image repo in registry
  image_repository: library
  # Kubespray container image for deploy user cluster or scale
  kubespray_image: "kubespray"
  # Auto generate self-signed certificate for registry domain
  generate_domain_crt: true
  # For nodes pull image, use 443 as default
  registry_https_port: 443
  # For push image to this registry, use 5000 as default, and only bind at 127.0.0.1
  registry_push_port: 5000
  # Set false to disable download all container images on all nodes
  download_container: false
```

### Deploy a new cluster

```bash
$ bash install.sh
```

### Add node to existing cluster

```bash
$ bash install.sh add-node $NODE_NAMES
```

### Delete node from cluster

```bash
$ bash install.sh remove-node $NODE_NAME
```

### Remove cluster

```bash
$ bash install.sh remove-cluster
```

### Remove all components

```bash
$ bash install.sh remove
```
