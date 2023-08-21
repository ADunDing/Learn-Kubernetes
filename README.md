# kubeadm安裝
SystemOS: Ubuntu 22.04

## 關閉SWAP
關閉SWAP
```
swapoff -a
```
檢查/etc/fstab設定
```
sudo vim /etc/fstab
```

確認swap相關設定都被註解掉
例如最後一行#/swap.img設定
```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-tJkzLnIcNn9IpZ0NwdcQB3jga8JFi1WGpdUa8E8EsyLIjX2EDxW71EabBZbgxhOh / ext4 defaults 0 1
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/17ca2aaa-91e9-4353-a605-babae10dd8ec /boot ext4 defaults 0 1
#/swap.img      none    swap    sw      0       0
```
改完以後重開機設定才會生效，使用指令```sudo reboot```重開機

## 檢查Port 6443

先檢查port 6443有沒有被佔用
```
nc -v 127.0.0.1 6443

nc: connect to 127.0.0.1 port 6443 (tcp) failed: Connection refused
```
Connection refused 代表沒有被佔用可以開始下一步
相反的，如果是 succeeded 代表有其他服務正在使用
```
Connection to 127.0.0.1 6443 port [tcp/*] succeeded!
```
可以使用 netstat 查看是誰在用，如果沒有裝 netstat 可改用 lsof
```
sudo netstat -tulpn | grep :6443

sudo lsof -i -P -n | grep :6443
```

## 安裝Docker
參考官網
> https://docs.docker.com/engine/install/ubuntu/
> 
**如果你是用snap套件管理系統安裝Docker的話建議刪除重裝**
**在安裝時順便安裝的docker就是使用snap安裝的**

將自己加入 Docker group
```
sudo usermod -aG docker $USER
```
加入後可不用每次都使用sudo叫用docker，直接使用docker即可

## 設定cgroup
使用`docker info | grep Cgroup`檢查
Cgroup Driver 是不是systemd
```
Cgroup Driver: systemd
Cgroup Version: 2
```
`Cgroup Driver: cgroupfs` 表示不正確
使用以下指令設定為systemd
```
sudo bash -c "cat > /etc/docker/daemon.json <<EOF
{
  \"exec-opts\": [\"native.cgroupdriver=systemd\"]
}
EOF
"
```
然後重啟docker
```
sudo systemctl restart docker
```
接著重複上面的檢查

## 安裝 cri-dockerd

>Kubernetes 已移除對 dockershim 的支援，若要繼續使用 Docker 作為 Container Runtime 需要安裝 cri-dockerd 做為介接 Kubernetes 和 Docker 的橋樑

> 參考cri-dockerd的github
> https://github.com/Mirantis/cri-dockerd#build-and-install
> Ubuntu有release可以下載
> https://github.com/Mirantis/cri-dockerd/releases

找到Ubuntu22.04(代號jammy)下載它`cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb`
```
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb
```
安裝cri-dockerd
```
sudo dpkg -i cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb
```
檢查版本
```
cri-dockerd --version
```
reload service
```
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```
檢查 service 狀態
```
sudo systemctl status cri-docker.socket
```
```
● cri-docker.socket - CRI Docker Socket for the API
     Loaded: loaded (/lib/systemd/system/cri-docker.socket; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-07-24 06:23:07 UTC; 3 weeks 5 days ago
   Triggers: ● cri-docker.service
     Listen: /run/cri-dockerd.sock (Stream)
      Tasks: 0 (limit: 19050)
     Memory: 0B
        CPU: 2ms
     CGroup: /system.slice/cri-docker.socket

Jul 24 06:23:07 88-123 systemd[1]: Starting CRI Docker Socket for the API...
Jul 24 06:23:07 88-123 systemd[1]: Listening on CRI Docker Socket for the API.
```
## 安裝Kubeadm
> 參考Kubernetes官方文件
> https://kubernetes.io/docs/setup/productionenvironment/tools/kubeadm/install-kubeadm/

1. 更新apt包索引並安裝使用 Kubernetesapt存儲庫所需的包：
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl
```
2. 下載 Kubernetes 包存儲庫的公共簽名密鑰
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```
3. 添加適當的 Kubernetesapt存儲庫
```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
4. 更新apt包索引，安裝 kubelet、kubeadm 和 kubectl，並固定它們的版本
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
目前為止的作業在所有的node都要做一次
# 建立 Cluster
## Control Plane
> 參考官方文件
> https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/
> Kubernetes Components 參考
> https://kubernetes.io/docs/concepts/overview/components/

選擇一台node作為Control Plane輸入以下指令
```
sudo kubeadm init \
    --apiserver-advertise-address=192.168.x.x \
    --pod-network-cidr=10.244.0.0/16 \
    --cri-socket /var/run/cri-dockerd.sock \
    --service-cidr=10.244.0.0/18
```
* `--apiserver-advertise-address`: 填Control Plane主機 ip，要與其他 node 在同個網段
* `--pod-network-cidr`: 填 cluster 內部使用的網段，**不能與主機網段衝突**
* `--cri-socket`: 填使用的 Container Runtime Interface (CRI)，上一篇有先安裝了 cri-dockerd 這邊需要自己指定，否則會因為偵測到多個 CRI 而失敗
* `--service-cidr`: 劃分給 service 的網段

最後一段顯示 cluster ca 和 token 之後 node 會需要使用它才能加入 cluster
```
kubeadm join 192.168.x.x:6443 --token r1dnp... \
        --discovery-token-ca-cert-hash sha256:25c414e47...
```
如果要讓 non-root user 操作 cluster 需要複製一份 root 的憑證資訊到 user 自己的 home 目錄下
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Worker Node
參考前面 Control Plane 完成 init 時產生的 kubeadm join 指令拿來執行，注意要補上 --cri-socket
```
sudo kubeadm join 192.168.x.x:6443 --token r1dnp... \
        --discovery-token-ca-cert-hash sha256:25c414e47... \
        --cri-socket unix:///var/run/cri-dockerd.sock
```
如果忘記 CA hash 可以回到 Control Plane 上用 openssl 指令取出來
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
如果 token 也忘了一樣可以回到 Control Plane 用指令找
```
sudo kubeadm token list
```
如果 token 已經過期了重新產生一組
```
sudo kubeadm token create --print-join-command
```
回到 Control Plane Plane 查看 node狀態
```
kubectl get nodes
```
```
NAME     STATUS   ROLES           AGE    VERSION
Control  NotReady control-plane   5d6h   v1.27.4
Worker1  NotReady <none>          5d6h   v1.27.4
Worker2  NotReady <none>          5d6h   v1.27.4
Worker3  NotReady <none>          5d6h   v1.27.4
```
## CNI (Container Network Interface) plugin
到目前為止的安裝每個 node 都會顯示 NotReady
```
kubectl get nodes
```
```
NAME     STATUS   ROLES           AGE    VERSION
Control  NotReady control-plane   5d6h   v1.27.4
Worker1  NotReady <none>          5d6h   v1.27.4
Worker2  NotReady <none>          5d6h   v1.27.4
Worker3  NotReady <none>          5d6h   v1.27.4
```
這時候需要安裝 network plugin
比較常見的CNI plugin為Flannel與Calico
安裝前先確認 kubeadm init 的參數 pod-network-cidr
```
kubectl cluster-info dump | grep -m 1 cluster-cidr
```
"--cluster-cidr=10.244.0.0/16",

### Flannel
> 參考Flannel github
> https://github.com/flannel-io/flannel

* Flannel 預設的 pod-network-cidr 為 10.244.0.0/16，如果不同須改 kube-flannel.yml
* 下載yaml檔
```
wget https://github.com/flannel-io/flannel/releases/download/v0.22.2/kube-flannel.yml
```
* 找到 ConfigMap(kube-flannel-cfg) > data > net-conf.json

使用kubectl部署
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

# Command line tool
> 常用指令介紹
> 全部指令參考官方
> https://kubernetes.io/docs/reference/kubectl/

### 查看pod運行在哪個node上
```
kubectl get pod -A \
    -o custom-columns=POD:metadata.name,Node:spec.nodeName \
    --sort-by spec.nodeName
```
### 透過 api 查看健康度
下面兩個指令是一樣的
```
kubectl get --raw='/readyz?verbose
```
```
curl -k https://localhost:6443/livez?verbose
```
### 查看node細節
```
kubectl describe node <nodeName>
```