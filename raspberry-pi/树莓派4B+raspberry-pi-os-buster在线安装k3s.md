# 树莓派4B+raspberry-pi-os-buster在线安装k3s

## 安装docker

参考：https://liujinye.gitbook.io/openshift-docs/raspberry-pi/shu-mei-pai-raspberrypios32bit-an-zhuang-docker

## 永久禁用swap

```
systemctl disable dphys-swapfile.service --now
```

## 在 Raspbian Buster 上启用旧版的 iptables
Raspbian Buster 默认使用nftables而不是iptables。 K3S 网络功能需要使用iptables，而不能使用nftables。 按照以下步骤切换配置Buster使用legacy iptables：

```
sudo iptables -F
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

编辑 `/boot/cmdline.txt`, 最后添加`cgroup_memory=1 cgroup_enable=memory`

完整内容如下：

```
console=serial0,115200 console=tty1 root=PARTUUID=ffd08aef-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait cgroup_memory=1 cgroup_enable=memory

```

重启主机

```
sudo reboot
```

## 安装k3s server

安装server有node相关参数是因为server内置了agent

```
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - --docker --node-name raspberrypi
```

密码列表

```
/var/lib/rancher/k3s/server/cred/passwd 
```

## 安装配置heml

### 安装 snaps

```
sudo apt update
sudo apt install snapd
```

### 安装heml

```
sudo snap install helm --classic
```

### 配置kube cofnig

复制k3s.yaml到 ~/.kube/config, 然后就helm 命令就可以管理了

```
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

```
helm ls --all-namespaces
```

## Kubernetes 仪表盘

### 部署 Kubernetes 仪表盘

```
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
sudo k3s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
```

树莓派环境在线安装 dashboard 会报证书错误

```
Unable to connect to the server: x509: certificate has expired or is not yet valid
```

在客户端电脑直接下载 `recommended.yaml` 文件，上传到树莓派主机再创建

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

本地文件安装 dashboard

```
k3s kubectl create -f recommended.yaml
```

### 仪表盘 RBAC 配置

创建以下资源清单文件：

dashboard.admin-user.yml

```
cat > dashboard.admin-user.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```
dashboard.admin-user-role.yml

```
cat > dashboard.admin-user-role.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
EOF
```

部署admin-user 配置：

```
sudo k3s kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml
```

### 端口转发仪表盘

```
kubectl port-forward --address 0.0.0.0 svc/kubernetes-dashboard 8443:443 -n kubernetes-dashboard
```

### 获取dashboard 管理token

sudo k3s kubectl -n kubernetes-dashboard describe secret admin-user-token | grep token

### 访问仪表盘

```
http://192.168.0.105:8443
```

chrome 浏览器提示报错 您的连接不是私密连接 NET::ERR_CERT_INVALID 相关错误，并且不能点击继续

在chrome该页面上，直接键盘敲入这11个字符：`thisisunsafe`

（鼠标点击当前页面任意位置，让页面处于最上层即可输入）

## 添加Agent

注意：所有节点的主机名都要不同

agent主机 添加命令示例，其中K3S_TOKEN ，是在从master主机`/var/lib/rancher/k3s/server/node-token` 获取的

```
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=<K3S_URL> K3S_TOKEN=<cat /var/lib/rancher/k3s/server/node-token> sh -s  - --docker --node-name <NODE_NAME> 
```

常用添加agent 参数

```
     --server value \
     --docker \
     --node-name value \
     --node-label foo=bar \
     --node-label hello=world \
     --node-taint key1=value1:NoExecute
```

最终示例：

```
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.0.105:6443 K3S_TOKEN=K10685e1ca93e1a980e14c2465a76e6217dafb1b4dc651793ce13a40b63a3a8c51a::server:63d408b61cec11e4fb891838b188764f sh -s  - --docker --node-name node107 --node-label bfs=true --node-label rnode=true
```

## 参考

https://docs.rancher.cn/docs/k3s/advanced/_index/#%E5%9C%A8-raspbian-buster-%E4%B8%8A%E5%90%AF%E7%94%A8%E6%97%A7%E7%89%88%E7%9A%84-iptables

https://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index

https://docs.rancher.cn/docs/k3s/installation/install-options/_index

https://liujinye.gitbook.io/openshift-docs/troubleshooting/macoschrome-fang-wen-https-ye-mian-xian-shi-errcertinvalid-qie-bu-neng-dian-ji-xu

https://snapcraft.io/install/helm/raspbian#install