### Giới thiệu

- Kubernetes thực hiện auto-scale số lượng pod dựa vào các metrics( cpu, ram) mà nó thu thập từ các pods. K8s sẽ đọc các thông số về cpu, ram của pods để quyết định có nên tạo thêm các pods mới không.

### Các bước thực hiện

- Bước 1: Cấu hình metric-server cho k8s để thực hiện thu thập các metrics của các pod trong k8s.

```
git clone https://github.com/kubernetes-incubator/metrics-server.git
kubectl create -f deploy/1.8+/
```

- Bước 2: Nếu triển khai k8s trên AWS bởi EKS hoặc Google GKE sẽ không bị gặp lỗi `Metrics server issue with hostname resolution of kubelet and apiserver unable to communicate with metric-server clusterIP`, nhưng dùng kubeadm để cài đặt trên bare metal sẽ gặp lỗi này, ta cần edit lại file metrics-server-deployment.yaml.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: k8s.gcr.io/metrics-server-amd64:v0.3.1
        command:
          - /metrics-server
          - --metric-resolution=5s
          - --kubelet-preferred-address-types=InternalIP
          - --kubelet-insecure-tls
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```

- Bước 3: Deploy metrics-server

```
kubectl apply -f deploy/1.8+/
```

- Bước 4: Check pod metrics-server

```
kubectl get pod -n kube-system
metrics-server-7f4856754f-6lmcj         1/1     Running   1          26h
```

- Bước 5: Tạo file deployment-nginx-frontend.yaml

```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: frontend
  labels:
    app: myapp
    role: frontend
spec:
  selector:
    matchLabels:
      app: myapp
      role: frontend
  template: # create pods using pod definition in this template
    metadata:
      labels:
        app: myapp
        role: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          limits: 
            cpu: "200m"
			memory: "200Mi"
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-frontend
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 90
```

- Bước 6: Apply file deployment-nginx-frontend.yaml

```
kubectl apply -f deployment-nginx-frontend.yaml
```

- Bước 7: Check HPA(horizontal-pod-autoscale) đã được tạo thành công chưa

```
kubectl get hpa
NAME           REFERENCE             TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
hpa-frontend   Deployment/frontend   2%/80%, 1%/90%    3         10        2          48d
```

- Bước 8: Tạo service nginx-frontend.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
  #  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  selector:
    app: myapp
    role: frontend
```

- Bước 9: Test service nginx-frontend đã được tạo thành công chưa

```
curl -s -I http://172.16.68.210:30080 | grep HTTP
HTTP/1.1 200 OK
```

- Bước 10: Cài đặt tool ab (apache benchmark) để giả lập request gửi đến http://172.16.68.210:30080

```
apt install apache2-utils
```

- Bước 11: Check HPA(horizontal-pod-autoscale) trước khi thực hiện gửi request tới http://172.16.68.210:30080, ta thấy số lượng Pod hiện tại chạy service nginx là 3

```
watch -n 1 kubectl get hpa
NAME           REFERENCE             TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
hpa-frontend   Deployment/frontend   2%/80%, 1%/90%    3         10        3          42d
```

- Bước 12: Thực hiện test gửi request đến http://172.16.68.210:30080/

```
ab -c 100 -n 500000 -t 100000 http://172.16.68.210:30080/
```

- Ta cũng có thể test ram/cpu trực tiếp của các pod với lệnh `stress`.

```
kubectl exec -it frontend-68cc5f445-95d8k /bin/bash
apt update
apt install stress
stress --vm 2 --vm-bytes 300M
```

- Bước 13: Check RAM/CPU của các pods với lệnh watch:

```
watch -n 1 kubectl get hpa
NAME           REFERENCE             TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
hpa-frontend   Deployment/frontend   85%/80%, 99%/90%    3         10        5          42d
```

- Bước 14: Check số lượng pod sau khi test, ta thấy số lượng Pod đã tăng lên thành 5.

```
kubectl get pod
frontend-68cc5f445-95d8k   1/1     Running   1          27h
frontend-68cc5f445-g4bss   1/1     Running   0          46s
frontend-68cc5f445-m8qwb   1/1     Running   0          27h
frontend-68cc5f445-pljcb   1/1     Running   0          46s
frontend-68cc5f445-qb4mk   1/1     Running   0          27h
```