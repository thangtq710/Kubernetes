## Tài liệu hướng dẫn cài đặt, cấu hình HA k8s cluster.

### 1. Mô hình:

![alt](../images/ha-cluster-k8s-topology.png)

- Trong mô hình này sẽ triển khai 1 cụm HA cluster k8s với external etcd.

### 2. Môi trường cài đặt

- 3 Ubuntu servers (18.04)

- 172.16.68.210: k8s-master-1 + etcd0 + worker1 + Haproxy1
- 172.16.68.211: k8s-master-2 + etcd1 + worker2 + Haproxy2
- 172.16.68.212: k8s-master-3 + etcd2 + worker3

- VIP 172.16.68.215 (Haproxy-1 + HAproxy-2)

- Root privileges

### 3. Các bước cần thực hiện:

- Cài đặt, cấu hình Haproxy + keepalived
- Cài đặt, cấu hình HA Etcd cluster
- Cài đặt, cấu hình masters nodes k8s và workers nodes k8s

#### Cài đặt, cấu hình Haproxy + keepalived

- Thực hiện trên 2 node 172.16.68.210, 172.16.68.211:

- Enable VIP:

  ```
  vim /etc/sysctl.conf

  net.ipv4.ip_nonlocal_bind=1
  
  sysctl -p
  ```
  
- Install Haproxy, keepalived:

  ```
  apt install keepalived -y && apt-get install -y haproxy
  ```

- Setup keepalived configuration:

  ```
  sudo mkdir -p /etc/keepalived

  cat > /etc/keepalived/keepalived.conf <<EOF
 
  vrrp_script check_haproxy {
  script "killall -0 haproxy"
  interval 3
  weight 3
  }

  vrrp_instance LAN_133 {
  interface ens33
  virtual_router_id 133
  priority 133
  advert_int 2
  
  authentication {
    auth_type PASS
    auth_pass ahihi
  }

  track_script {
    check_haproxy
  }
 
  virtual_ipaddress {
    172.16.68.215/24
    }
  }

  EOF
  ```

- Setup Haproxy

```
cat > /etc/haproxy/haproxy.cfg <<EOF
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend k8s
bind 172.16.68.215:6445
option tcplog
mode tcp
default_backend k8s-master-nodes


backend k8s-master-nodes
mode tcp
balance roundrobin
option tcp-check
server k8s-master-1 172.16.68.210:6443 check fall 3 rise 2
server k8s-master-2 172.16.68.211:6443 check fall 3 rise 2
server k8s-master-3 172.16.68.212:6443 check fall 3 rise 2

EOF
```

- Thực hiện restart haproxy và keepalived:

  ```
  systemctl restart haproxy
  systemctl restart keepalived
  ```

- Kiểm tra:

  ```
  ip a | grep ens3
  ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 172.16.68.210/24 brd 172.16.68.255 scope global ens3
    inet 172.16.68.215/24 brd 172.16.68.255 scope global secondary ens3:1
  ```

#### Cài đặt, cấu hình HA Etcd cluster:

- https://github.com/thangtq710/Kubernetes/blob/master/docs/timhieu-etcd.md

#### Cài đặt, cấu hình master node k8s và worker node k8s:  

#### Thực hiện trên 3 node master và 3 node worker:

- Disable swap:
  
  ```
  swapoff -a
  ```

- Install Kubeadm Packages:
  
  ```
  apt update -y && apt upgrade -y
  
  apt-get install apt-transport-https curl -y
  
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
  
  vim /etc/apt/sources.list.d/kubernetes.list
  deb http://apt.kubernetes.io/ kubernetes-xenial main
  
  apt update
  apt install -y kubeadm kubelet kubectl
  ```
  
- Install Docker:

  ```
  apt-get install docker.io -y
  ```

- Edit Docker:
  
  ```
  modprobe overlay
  modprobe br_netfilter
  
  cat > /etc/docker/daemon.json <<EOF
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2"
  }
  EOF
  
  systemctl daemon-reload
  systemctl restart docker
  ```
  
- Config kubernetes-cri:

  ```
  cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
  net.bridge.bridge-nf-call-iptables  = 1
  net.ipv4.ip_forward                 = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  EOF
  
  sysctl --system
  ```

#### Thực hiện trên node master 1:

- Copy cert to /etc/kubernetes/pki:
  
  ```
  mkdir -p /etc/kubernetes/pki/etcd
  
  cp /etc/etcd/ca.pem /etc/kubernetes/pki/etcd/
  
  cp /etc/etcd/kubernetes.pem /etc/kubernetes/pki/
  
  cp /etc/etcd/kubernetes-key.pem /etc/kubernetes/pki/
  ```

- Create file kubeadm-config.yaml trên node master-1 (etcd0):

  ```
  apiVersion: kubeadm.k8s.io/v1beta1
  kind: ClusterConfiguration
  kubernetesVersion: stable
  controlPlaneEndpoint: "172.16.68.215:6445"
  etcd:
      external:
          endpoints:
          - https://172.16.68.210:2379
          - https://172.16.68.211:2379
          - https://172.16.68.212:2379
          caFile: /etc/kubernetes/pki/etcd/ca.pem
          certFile: /etc/kubernetes/pki/kubernetes.pem
          keyFile: /etc/kubernetes/pki/kubernetes-key.pem
  networking:
      podSubnet: "10.244.0.0/16"
  ```

- Thực hiện khởi tạo node master đầu tiên trong cụm k8s:

```
master1# kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs
```

- Trong output ở trên có các dòng sau, sử dụng để join các node master và các node worker vào cụm cluster ở phần dưới:
   
   ```
   kubeadm join 172.16.68.215:6445 --token dzvrxa.3m8ajhpiycj3btek     --discovery-token-ca-cert-hash sha256:d770d2a3e3494fcc090641de61     --experimental-control-plane --certificate-key 635054d25d49336bdd9022ceebe447
   
   kubeadm join 172.16.68.215:6445 --token dzvrxa.3m8ajhpiycj3btek --discovery-token-ca-cert-hash sha256:d770d2a3e3494fcc090641de61e005b2b6d8
   ```

- Khởi tạo môi trường:

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

- Triển khai `flannel` network đến cụm k8s:

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

- Join các node master vào cụm k8s, thực hiện trên 2 node master2 và master3:

```
kubeadm join 172.16.68.215:6445 --token dzvrxa.3m8ajhpiycj3btek     --discovery-token-ca-cert-hash sha256:d770d2a3e3494fcc090641de61     --experimental-control-plane --certificate-key 635054d25d49336bdd9022ceebe44

mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

chown $(id -u):$(id -g) $HOME/.kube/config

```

- Để thực hiện 1 node vừa làm master và worker, ta cần thực hiện lệnh sau:

 ```
 kubectl taint nodes --all node-role.kubernetes.io/master-
 
 kubectl get node -o wide
 NAME       STATUS   ROLES    AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
 master-1   Ready    master   5d    v1.14.1   172.16.68.210   <none>        Ubuntu 18.04.2 LTS   4.15.0-50-generic   docker://18.9.2
 master-2   Ready    master   5d    v1.14.1   172.16.68.211   <none>        Ubuntu 18.04.2 LTS   4.15.0-48-generic   docker://18.9.2
 master-3   Ready    master   5d    v1.14.1   172.16.68.212   <none>        Ubuntu 18.04.2 LTS   4.15.0-50-generic   docker://18.9.2
 ```
 
- Check các pod trong namespace kube-system:

```
root@master-1:~# kubectl get pod -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-b2wwb            1/1     Running   0          79m
coredns-fb8b8dccf-z9d5h            1/1     Running   0          79m
kube-apiserver-master-1            1/1     Running   0          78m
kube-apiserver-master-2            1/1     Running   0          76m
kube-apiserver-master-3            1/1     Running   0          76m
kube-controller-manager-master-1   1/1     Running   0          78m
kube-controller-manager-master-2   1/1     Running   0          76m
kube-controller-manager-master-3   1/1     Running   0          76m
kube-flannel-ds-amd64-mbmhw        1/1     Running   0          76m
kube-flannel-ds-amd64-pc4gb        1/1     Running   0          12m
kube-flannel-ds-amd64-pfnr2        1/1     Running   0          77m
kube-flannel-ds-amd64-rgd2q        1/1     Running   0          76m
kube-flannel-ds-amd64-th5v5        1/1     Running   0          13m
kube-flannel-ds-amd64-ttkzk        1/1     Running   0          13m
kube-proxy-7nk98                   1/1     Running   0          13m
kube-proxy-8mtwq                   1/1     Running   0          13m
kube-proxy-bf5zd                   1/1     Running   0          76m
kube-proxy-h9h8f                   1/1     Running   0          12m
kube-proxy-z8wgf                   1/1     Running   0          76m
kube-proxy-zzt96                   1/1     Running   0          79m
kube-scheduler-master-1            1/1     Running   0          78m
kube-scheduler-master-2            1/1     Running   0          76m
kube-scheduler-master-3            1/1     Running   0          76m
```

- Nếu build cụm k8s với các node master và node worker tách biệt, thực hiện join các work node vào cụm k8s, ta thực hiện trên các worker node:

```
kubeadm join 172.16.68.215:6443 --token dzvrxa.3m8ajhpiycj3btek --discovery-token-ca-cert-hash sha256:d770d2a3e3494fcc090641de61e005b2b6d8
```

#### Chú ý:

- Mặc định, trong cụm k8s khi 1 node worker down thì thời gian các pods trên node đó tự động được tạo lại trên node worker khác là 5'. Ta có thể giảm thời gian này xuống, để các pods được tạo lại trên node worker khác nhanh hơn ( ở đây tôi set khoảng thời gian này = 1') bằng cách thực hiện các bước sau:

- Bước 1: Thêm `--pod-eviction-timeout=30s` vào file /etc/kubernetes/manifests/kube-controller-manager.yaml

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --bind-address=127.0.0.1
    - --pod-eviction-timeout=30s
	...
```

- Bước 2: Thêm `default-not-ready-toleration-seconds=30` và `default-unreachable-toleration-seconds=30` vào file /etc/kubernetes/manifests/kube-apiserver.yaml (thực hiện đối với k8s từ phiên bản từ 1.11 trở lên)

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=172.168.68.210
    - --default-not-ready-toleration-seconds=30
    - --default-unreachable-toleration-seconds=30
    - --allow-privileged=true
    ...
```

### 4. Link tham khảo

- https://kubernetes.io/docs/setup/independent/high-availability/

- https://blog.inkubate.io/install-and-configure-a-multi-master-kubernetes-cluster-with-kubeadm/

- https://github.com/kubernetes-sigs/kubespray/blob/master/docs/kubernetes-reliability.md
