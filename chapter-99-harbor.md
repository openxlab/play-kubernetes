# Chapter 99 : 部署harbor

## 基于CentOS部署Harbor

```bash
"E:\PUTTY\PUTTY.EXE" -ssh -C -X -P 22 -pw "root2018" root@192.168.80.100
ssh root@192.168.80.100


grep -v ^# /etc/yum.conf | grep -v ^$
sudo sed -i 's/keepcache=0/keepcache=1/' /etc/yum.conf
echo "proxy=http://proxy.com.cn:80" | sudo tee -a /etc/yum.conf
cat <<EOF | tee -a ~/.bashrc
alias curl='curl -x http://proxy.com.cn:80'
export http_proxy="http://proxy.com.cn:80"
export https_proxy="http://proxy.com.cn:80"
export no_proxy="localhost,127.0.0.1,192.168.80.100"
EOF
source ~/.bashrc

sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum list docker-ce --showduplicates | sort -r
sudo yum install -y docker-ce

sudo mkdir -p /etc/systemd/system/docker.service.d
cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://proxy.com.cn:80/"
Environment="NO_PROXY=localhost,127.0.0.1,192.168.80.100"
EOF
sudo systemctl daemon-reload
sudo systemctl show --property=Environment docker

sudo systemctl start docker

alias curl='curl -x http://proxy.com.cn:80'
curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version


curl -L -O https://storage.googleapis.com/harbor-releases/release-1.6.0/harbor-offline-installer-v1.6.0-rc1.tgz
tar xvf harbor-offline-installer-v1.6.0-rc1.tgz
cd harbor
grep -v ^# harbor.cfg | grep -v ^$
sed "s/^hostname = .*/hostname = 192.168.80.100/g" -i ./harbor.cfg
./install.sh
```

## 在docker client配置

```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "insecure-registries" : ["192.168.80.100"]
}
EOF
sudo systemctl restart docker

docker login 192.168.80.100
docker tag rancher/rancher:latest 192.168.80.100/openxlab/rancher:2.0
docker push 192.168.80.100/openxlab/rancher:2.0
```

