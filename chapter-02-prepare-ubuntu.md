# Chapter 02 : 准备Ubuntu

#### 安装Ubuntu操作系统

{% hint style="warning" %}
文件系统注意选择为"btrfs"。
{% endhint %}

#### 配置Ubuntu相关系统参数

```bash
ssh 'admuser@192.168.80.100'

sudo sed -i 's/192.168.80.199/192.168.80.100/' /etc/netplan/01-netcfg.yaml
echo "admuser ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/admuser
sudo chmod 0440 /etc/sudoers.d/admuser

sudo sed -i 's/\/dev\/mapper\/k8s--master--vg-swap_1/\#\/dev\/mapper\/k8s--master--vg-swap_1/' /etc/fstab

cat <<EOF | sudo tee /etc/hosts
127.0.0.1         k8s-master
192.168.80.100    k8s-master
EOF

cat <<EOF | sudo tee /etc/hostname
k8s-master
EOF

sudo reboot
```

#### 配置系统代理

```bash
alias curl='curl -x http://192.168.80.5:1090'
export http_proxy="http://192.168.80.5:1090"
export https_proxy="https://192.168.80.5:1090"
export no_proxy="localhost,127.0.0.1,172.17.0.0/16,192.168.80.100"

echo 'Acquire::http::Proxy "http://192.168.80.5:1090";' | sudo tee /etc/apt/apt.conf
```

#### 安装并配置Docker

```bash
ps -e | grep -e apt -e adept | grep -v grep
sudo apt-get remove -y --purge lxd-client lxcfs

sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo reboot

sudo apt-get install -y curl gnupg software-properties-common apt-transport-https ca-certificates
curl -x http://192.168.80.5:1090 -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update

sudo apt-cache madison docker-ce
sudo apt-get install -y docker-ce
sudo usermod -a -G docker $USER
sudo systemctl enable docker && sudo systemctl start docker
```

#### 采用kubeadm部署Kubernetes

```bash
curl -x http://192.168.80.5:1090 -s https://packages.cloud.google.com/apt/doc/apt-key.gpg |sudo apt-key add -
sudo add-apt-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://192.168.80.5:1090/"
Environment="HTTPS_PROXY=http://192.168.80.5:1090/"
Environment="NO_PROXY=127.0.0.1,172.17.0.0/16,192.168.80.100"
EOF

sudo systemctl daemon-reload
sudo systemctl show --property=Environment docker
sudo systemctl restart docker

sudo kubeadm config images pull --v=4
sudo kubeadm init \
  --kubernetes-version=v1.11.2 \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.80.100
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl get pods --all-namespaces
kubectl get services --all-namespaces
kubectl get nodes -o wide
kubectl cluster-info

curl -x http://192.168.80.5:1090 -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
curl -x http://192.168.80.5:1090 -O https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl apply -f kubernetes-dashboard.yaml

kubectl -n kube-system edit service kubernetes-dashboard
type: ClusterIP >> type: NodePort
kubectl -n kube-system get service kubernetes-dashboard
https://192.168.80.100:30436

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-admin
  namespace: kube-system
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kube-rolebinding-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kube-admin
  namespace: kube-system
EOF

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kube-admin | awk '{print $1}')

```

#### 配置Helm

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

curl -x http://192.168.80.5:1090 -O https://storage.googleapis.com/kubernetes-helm/helm-v2.10.0-linux-amd64.tar.gz
tar -zxvf helm-v2.10.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm && rm -rf linux-amd64 helm-v2.10.0-linux-amd64.tar.gz
helm init --service-account tiller
helm version
kubectl get pods --namespace kube-system
```

#### 配置minio

```bash
curl -L -O https://kubernetes-charts.storage.googleapis.com/minio-1.6.2.tgz
tar -zxvf minio-1.6.2.tgz
sed -i 's/type: ClusterIP/type: NodePort/' ./minio/values.yaml
sed -i 's/# nodePort: 31311/nodePort: 30900/' ./minio/values.yaml
helm install --name minio-162 --set accessKey=minio,secretKey=miniominio,persistence.enabled=false ./minio
helm delete --purge minio-162
```

#### 使用PV PVC部署minio

```yaml
mkdir -p /home/admuser/localpv
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: openxlab-local-pv-5gi
spec:
  capacity:
    storage: 5Gi 
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /home/admuser/localpv
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-master
EOF

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  # This name uniquely identifies the PVC. Will be used in deployment below.
  name: minio-pv-claim
  labels:
    app: minio-storage-claim
spec:
  # Read more about access modes here: http://kubernetes.io/docs/user-guide/persistent-volumes/#access-modes
  accessModes:
    - ReadWriteOnce
  resources:
    # This is the request for storage. Should be available in the cluster.
    requests:
      storage: 2Gi
  # Uncomment and add storageClass specific to your requirements below. Read more https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1
  storageClassName: local-storage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  # This name uniquely identifies the Deployment
  name: minio-deployment
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        # Label is used as selector in the service.
        app: minio
    spec:
      # Refer to the PVC created earlier
      volumes:
      - name: storage
        persistentVolumeClaim:
          # Name of the PVC created earlier
          claimName: minio-pv-claim
      containers:
      - name: minio
        # Pulls the default Minio image from Docker Hub
        image: minio/minio
        args:
        - server
        - /storage
        env:
        # Minio access key and secret key
        - name: MINIO_ACCESS_KEY
          value: "minio"
        - name: MINIO_SECRET_KEY
          value: "minio2018"
        ports:
        - containerPort: 9000
        # Mount the volume into the pod
        volumeMounts:
        - name: storage # must match the volume name, above
          mountPath: "/storage"
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
spec:
  type: NodePort
  ports:
    - port: 9000
      nodePort: 30900
      targetPort: 9000
      protocol: TCP
  selector:
    app: minio
EOF

```

