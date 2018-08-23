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
export no_proxy="localhost,127.0.0.1,192.168.80.100"

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

