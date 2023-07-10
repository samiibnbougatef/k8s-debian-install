# install docker 
<br>

 ```
 sudo apt -y install lsb-release gnupg apt-transport-https ca-certificates curl software-properties-common
 
 curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg

 sudo add-apt-repository "deb [arch=$(dpkg --print-architecture)] https://download.docker.com/linux/debian $(lsb_release -cs) stable"

 sudo apt update
 sudo apt install docker-ce docker-ce-cli containerd.io

 # if you want to run docker in regular user
 sudo usermod -aG docker $USER
 ```
# install cri-dockerd
<br>

```
sudo apt update
sudo apt install git wget curl

VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
echo $VER

wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz
tar xvf cri-dockerd-${VER}.amd64.tgz

sudo mv cri-dockerd/cri-dockerd /usr/local/bin/

# check version 
cri-dockerd --version

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service

sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
sudo systemctl start cri-docker.service
sudo systemctl start cri-docker.socket

```
# kubernetes Cluster v1.26.2
* Forwarding IPv4 and letting iptables see bridged traffic
  <br> 
  ```
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF

  sudo modprobe overlay
  sudo modprobe br_netfilter

  # sysctl params required by setup, params persist across reboots
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  # Apply sysctl params without reboot
  sudo sysctl --system
  ```
* Verify that the br_netfilter, overlay modules are loaded
  <br>
  ```
  lsmod | grep br_netfilter
  lsmod | grep overlay
  ```
* Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config
  <br>
  ```
  sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
  ```
* Disable Swap
  <br>
  ```
  sudo swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  ```

* install packages needed to use the Kubernetes
  <br>
  ```
  sudo apt-get update
  sudo apt-get install -y apt-transport-https ca-certificates curl
  ```

* Google Cloud public signing key
  <br>
  ```
  sudo curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -  
  ```
  
* Add the Kubernetes apt repository  
  <br>
  ```
  sudo apt-add-repository  "deb https://apt.kubernetes.io/ kubernetes-xenial main"
  ```

* Update apt package index, install kubelet, kubeadm and kubectl, and pin their version
  <br>
  ```
  sudo apt-get update
  sudo apt-get install -y kubelet=1.26.2-00 kubeadm=1.26.2-00 kubectl=1.26.2-00
  sudo apt-mark hold kubelet kubeadm kubectl
  ```
* Initializing your control-plane node
  <br>
  ```
  kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket /run/cri-dockerd.sock
  ```
* Adding worker
  <br>

  ```
  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket /run/cri-dockerd.sock
  ```  

  * Installing CNI
   <br>

   ```
   kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/canal.yaml 
   ```