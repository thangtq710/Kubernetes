### 1. Giới thiệu

- Etcd: là 1 cơ sở dữ liệu key-value phân tán, tạo bởi CoreO, nay được quản lý bởi Cloud Native Computing Foundation. Nó đóng vai trò là xương sống của nhiều hệ thống phân tán quy mô lớn.

- `Etcd` hoạt động trên nhiều hệ điều hành: Linux, BSD và OS X.
	
### 2. Cơ chế hoạt động của Etcd:

- Để hiểu cách `etcd` hoạt động, ta cần xác định 3 khái niệm trong `etcd`: `leaders`, `elections` và `terms`. 

- Mọi thay đổi trong `etcd` sẽ được chuyển đến node `leader`. Thay vì chấp nhận và cam kết thay đổi ngay lập tức, `etcd` sử dụng thuật toán Raft để đảm bảo rằng phần lớn các node trong cụm cluster đều đồng ý về thay đổi. Node `leader` gửi giá trị mới được đề xuất cho mỗi node trong cụm. Các node sau đó gửi 1 thông báo xác nhận là đã nhận giá trị mới. Nếu phần lớn các node đã xác nhận, `leader` cam kết giá trị mới và thông báo cho mỗi node rằng giá trị được cam kết với nhật ký. Điều này có nghĩa là mỗi thay đổi đòi hỏi một đại biểu từ các node trong cụm để được cam kết.

- Nếu trong 1 khoảng thời gian node `leader` không còn phản hồi, các node còn lại sẽ bắt đầu 1 cuộc bầu cử mới (`elections`) để chọn ra 1 `leader` mới. Dựa trên thuật toán Raft, trong 1 cụm cluster etcd, các node sẽ thực hiện bầu cử để chọn ra 1 `leaders`.


### 3. Etcd trong Kubernetes

- `Etcd` được biết đến như là kho dữ liệu chính của k8s, được sử dụng để lưu trữ dữ liệu cấu hình, trạng thái và metadata trong k8s.

- `Etcd` theo dõi các thay đổi trong cụm k8s, cho phép mọi node master từ cụm k8s đọc và ghi dữ liệu. Chức năng `watch` của `Etcd` được k8s sử dụng để theo dõi các thay đổi về trạng thái thực tế hoặc trạng thái mong muốn của hệ thống. Nếu chúng khác nhau, k8s thực hiện các thay đổi để dung hòa 2 trạng thái này.

- Mỗi lần thực hiện đọc bởi lệnh `kubectl`, dữ liệu sẽ được lấy trong `etcd`, mọi thay đổi được thực hiện ( `kubectl apply`) sẽ tạo hoặc cập nhập các mục vào trong `etcd`.

### 4. Một số khuyến nghị khi triển khai cluster etcd trong thực tế.
	
- https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md

### 5. Lab sử dụng kubeadm để triển khai cụm cluster etcd

- Mô hình:

- 3 server ubuntu 18.04
	
	+ 172.16.68.210 etcd0
	
	+ 172.16.68.211 etcd1
	
	+ 172.16.68.212 etcd2

### Install cfssl (Cloudflare ssl) thực hiện trên node etcd0:

- Install and config:

  ```
  1. wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  
  2. wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  
  3. chmod +x cfssl*
  
  4. mv cfssl_linux-amd64 /usr/local/bin/cfssl
  
  5. mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
  ```

- Verify the installation:
  
  ```
  cfssl version
  ```
  
### Generating the TLS certificates:

- Tạo file cấu hình CA (certificate authority):

  ```
  vim ca-config.json
  {
    "signing": {
      "default": {
        "expiry": "8760h"
      },
      "profiles": {
        "kubernetes": {
          "usages": ["signing", "key encipherment", "server auth", "client auth"],
          "expiry": "8760h"
        }
      }
    }
  }
  ```
  
- Create the certificate authority signing request configuration file.
  
  ```
  vim ca-csr.json
  {
    "CN": "Kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
    {
      "C": "IE",
      "L": "VNPT",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "VNPT Co."
    }
   ]
  }
  ```

- Generate the certificate authority certificate and private key.

  ```
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
  ```

- Verify that the ca-key.pem and the ca.pem were generated.
  
  ```
  ll
  ```

### Creating the certificate for the Etcd cluster

- Create the certificate signing request configuration file.

  ```
  vim kubernetes-csr.json
  {
    "CN": "kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
    {
      "C": "IE",
      "L": "VNPT",
      "O": "Kubernetes",
      "OU": "Kubernetes",
      "ST": "VNPT Co."
    }
   ]
  }
  ```
- Generate the certificate and private key.

  ```
  cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.16.68.210,172.16.68.211,172.16.68.212,172.16.68.215,127.0.0.1,kubernetes.default \
  -profile=kubernetes kubernetes-csr.json | \
  cfssljson -bare kubernetes
  ```
  
- Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.

  ```
  ll
  ```

- Copy the certificate to each nodes on etcd cluster:

  ```
  scp ca.pem kubernetes.pem kubernetes-key.pem root@172.16.68.211:~
  
  scp ca.pem kubernetes.pem kubernetes-key.pem root@172.16.68.212:~
  ```

### Install and config Etcd cluster:

##### Trên node etcd0:

- 1.Create a configuration directory for Etcd.
  
  ```
  mkdir /etc/etcd /var/lib/etcd
  ```

- 2.Move the certificates to the configuration directory.

  ```
  mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd
  ```
  
- 3.Download the etcd binaries.

  ```
  wget https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz
  ```
  
- 4.Extract the etcd archive.

  ```
  tar xvzf etcd-v3.3.9-linux-amd64.tar.gz
  ```
  
- 5.Move the etcd binaries to /usr/local/bin.

  ```
  mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
  ```

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd0 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.210:2380 \
    --listen-peer-urls https://172.16.68.210:2380 \
    --listen-client-urls https://172.16.68.210:2379 \
    --advertise-client-urls https://172.16.68.210:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```
  
##### Trên node etcd1:

- Thực hiện các bước 1 -> 5 tương tự như trên node etcd0

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd1 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.211:2380 \
    --listen-peer-urls https://172.16.68.211:2380 \
    --listen-client-urls https://172.16.68.211:2379 \
    --advertise-client-urls https://172.16.68.211:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```

##### Trên node etcd2:

- Thực hiện các bước 1 -> 5 tương tự như trên node etcd0

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd2 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.212:2380 \
    --listen-peer-urls https://172.16.68.212:2380 \
    --listen-client-urls https://172.16.68.212:2379 \
    --advertise-client-urls https://172.16.68.212:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```

##### Trên cả 3 node etcd:

- Reload the daemon configuration.
  
  ```
  systemctl daemon-reload
  ```
  
- Enable etcd to start at boot time.
  
  ```
  systemctl enable etcd
  ```
  
- Start etcd.

  ```
  systemctl start etcd
  ```

- Verify etcd-cluster:
  
  ```
  ETCDCTL_API=3 etcdctl --write-out=table member list
  +------------------+---------+-------+----------------------------+----------------------------+
  |        ID        | STATUS  | NAME  |         PEER ADDRS         |        CLIENT ADDRS        |
  +------------------+---------+-------+----------------------------+----------------------------+
  | ab84f8691a2c742c | started | etcd0 | https://172.16.68.210:2380 | https://172.16.68.210:2379 |
  | e1a8c0925289a69d | started | etcd1 | https://172.16.68.211:2380 | https://172.16.68.211:2379 |
  | f6a2daf825ab64d5 | started | etcd2 | https://172.16.68.212:2380 | https://172.16.68.212:2379 |
  +------------------+---------+-------+----------------------------+----------------------------+

  
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.210:2379,https://172.16.68.211:2379,https://172.16.68.212:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/kubernetes.pem --key=/etc/etcd/kubernetes-key.pem --write-out=table endpoint status
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  |          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  | https://172.16.68.210:2379 | ab84f8691a2c742c |   3.3.9 |  1.9 MB |      true |         6 |       8213 |
  | https://172.16.68.211:2379 | e1a8c0925289a69d |   3.3.9 |  1.9 MB |     false |         6 |       8213 |
  | https://172.16.68.212:2379 | f6a2daf825ab64d5 |   3.3.9 |  1.9 MB |     false |         6 |       8213 |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  ```
- List key on etcd ( sau khi kết nối tới node-master trong k8s )

  ```
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.210:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only
  ```

### 6. Tham khảo:

- https://raft.github.io/
- https://rancher.com/blog/2019/2019-01-29-what-is-etcd/
- https://github.com/etcd-io/etcd/tree/master/Documentation
- https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md



