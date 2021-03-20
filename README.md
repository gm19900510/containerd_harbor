# 一、前言
`Kubernetes` 在 `Changelog` 中宣布自 `Kubernetes 1.20 之后将弃用 Docker 作为容器运行`时之后，`containerd` 成为下一个容器运行时的热门选项。虽然 `containerd` 很早就已经是 `Docker` 的一部分，但是纯粹使用 containerd 还是给大家带来了诸多困扰，本文将介绍如何使用 `containerd` 配置镜像仓库和加速器。

本文将以 `K3s` 为例对 `containerd` 进行配置，如果您的环境未使用 `K3s` 而是使用的 `Kubernetes`，你也可以参考本文来配置 `containerd` 的镜像仓库，因为 `containerd` 的配置是通用的。

# 二、关于 K3s 和 containerd
`K3s` 是一个轻量级 `Kubernetes` 发行版，二进制大小小于 `100MB`，所需内存不到 `Kubernetes` 的一半。`K3s` 为了降低资源消耗，将默认的 `runtime` 修改为 `containerd`，同时也内置了 `Kubernetes CLI` 工具 `crictl` 和 `ctr`。

`K3s` 默认的 `containerd` 配置文件目录为 `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`，但直接操作 `containerd` 的配置文件去设置镜像仓库或加速器相比于操作 `docker` 要复杂许多。`K3s` 为了简化配置 `containerd` 镜像仓库的复杂度，`K3s` 会在启动时检查`/etc/rancher/k3s/`中是否存在 `registries.yaml` 文件，如果存在该文件，就会根据 `registries.yaml` 的内容转换为 `containerd` 的配置并存储到 `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`，从而降低了配置 `containerd` 镜像仓库的复杂度。

# 三、使用 K3s 配置私有镜像仓库
`K3s` 镜像仓库配置文件由两大部分组成：`mirrors` 和 `configs`:

- `Mirrors` 是一个用于定义专用镜像仓库的名称和 `endpoint` 的指令
- `Configs` 部分定义了每个 `mirror` 的 `TLS` 和证书配置。对于每个 `mirror`，你可以定义 `auth` 和`/`或 `tls`

containerd 使用了类似 K8S 中 svc 与 endpoint 的概念，svc 可以理解为访问名称，这个名称会解析到对应的 endpoint 上。也可以理解 mirror 配置就是一个反向代理，它把客户端的请求代理到 endpoint 配置的后端镜像仓库。mirror 名称可以随意填写，但是必须符合 IP 或域名的定义规则。并且可以配置多个 endpoint，默认解析到第一个 endpoint，如果第一个 endpoint 没有返回数据，则自动切换到第二个 endpoint，以此类推。
## 3.1 非安全（http）私有仓库配置
### 3.1.1 有认证
如果你的非安全（http）私有仓库带有认证，那么可以通过下面的参数来配置 k3s 连接私有仓库：
```bash
[root@master ~]# cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "192.168.3.128:8999":
    endpoint:
      - "http://192.168.3.128:8999"
configs:
  "192.168.3.128:8999":
    auth:
      username: admin # this is the registry username
      password: Harbor12345 # this is the registry password

EOF
[root@master ~]# systemctl restart k3s
```

通过 crictl 去 pull 镜像

```bash
[root@master ~]# crictl pull 192.168.3.128:8999/private/hello-world:latest
Image is up to date for sha256:bf756fb1ae65adf866bd8c456593cd24beb6a0a061dedf42b26a993176745f6b
[root@master ~]# crictl images
IMAGE                                      TAG                 IMAGE ID            SIZE
192.168.3.128:8999/private/hello-world     latest              bf756fb1ae65a       4.56kB
```
查看Containerd 配置文件末尾追加的配置

```bash
[root@master ~]# cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml

[plugins.opt]
  path = "/var/lib/rancher/k3s/agent/containerd"

[plugins.cri]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  enable_selinux = false
  sandbox_image = "docker.io/rancher/pause:3.1"

[plugins.cri.containerd]
  disable_snapshot_annotations = true
  snapshotter = "overlayfs"

[plugins.cri.cni]
  bin_dir = "/var/lib/rancher/k3s/data/a6857be08414815b83ca6b960373efd98879a0b286fb24cb62b1c5fdbf3a8cb5/bin"
  conf_dir = "/var/lib/rancher/k3s/agent/etc/cni/net.d"


[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"



[plugins.cri.registry.mirrors]

[plugins.cri.registry.mirrors."192.168.3.128:8999"]
  endpoint = ["http://192.168.3.128:8999"]




[plugins.cri.registry.configs."192.168.3.128:8999".auth]
  username = "admin"
  password = "Harbor12345"

```

### 3.1.2 无认证
参照上文有证人部分去掉

```bash
auth:
  username: admin # this is the registry username
  password: Harbor12345 # this is the registry password
```
部分即可
## 3.2 安全（https）私有仓库配置
### 3.2.1 使用自签 ssl 证书
如果后端仓库使用的是自签名的 ssl 证书，那么需要配置 CA 证书 用于 ssl 证书的校验

```bash
[root@master ~]# cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "www.harbor.mobi:1443":
    endpoint:
      - "https://www.harbor.mobi:1443"
configs:
  "www.harbor.mobi:1443":
    auth:
      username: admin # this is the registry username
      password: Harbor12345 # this is the registry password
    tls:
      ca_file: /etc/pki/ca-trust/source/anchors/www.harbor.mobi.crt
EOF
[root@master ~]# systemctl restart k3s
[root@master ~]# update-ca-trust force-enable
[root@master ~]# update-ca-trust extract
```
> 其中`ca_file: /etc/pki/ca-trust/source/anchors/www.harbor.mobi.crt`是生产的Harbor证书文件。

通过 crictl 去 pull 镜像

```bash
[root@master ~]# crictl pull www.harbor.mobi:1443/private/whoami:latest
Image is up to date for sha256:0f6fbbedd3777530ea3bedadf0a75b9aba805a55f6c5481ef0ebd762c5eeb818
[root@master ~]# 

查看Containerd 配置文件末尾追加的配置

```bash
[root@master ~]# cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml

[plugins.opt]
  path = "/var/lib/rancher/k3s/agent/containerd"

[plugins.cri]
  stream_server_address = "127.0.0.1"
  stream_server_port = "10010"
  enable_selinux = false
  sandbox_image = "docker.io/rancher/pause:3.1"

[plugins.cri.containerd]
  disable_snapshot_annotations = true
  snapshotter = "overlayfs"

[plugins.cri.cni]
  bin_dir = "/var/lib/rancher/k3s/data/a6857be08414815b83ca6b960373efd98879a0b286fb24cb62b1c5fdbf3a8cb5/bin"
  conf_dir = "/var/lib/rancher/k3s/agent/etc/cni/net.d"


[plugins.cri.containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"



[plugins.cri.registry.mirrors]

[plugins.cri.registry.mirrors."www.harbor.mobi:1443"]
  endpoint = ["https://www.harbor.mobi:1443"]




[plugins.cri.registry.configs."www.harbor.mobi:1443".auth]
  username = "admin"
  password = "Harbor12345"
  
  


[plugins.cri.registry.configs."www.harbor.mobi:1443".tls]
  ca_file = "/etc/pki/ca-trust/source/anchors/www.harbor.mobi.crt"
  
  
```
https://my.oschina.net/u/4148359/blog/4885268
