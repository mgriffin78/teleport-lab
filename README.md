# Teleport Lab
This lab is used to demonstrate a Kubernetes cluster with an NGINX application. Certificates are managed via Cert Manager and access is controlled through Teleport. 

# Components used in this cluster

   - Kubernetes (Deployed via kubeadm)
   - Helm - Package manager used to deploy control plane elements and applications
   - Cert Manager - Manages certificates for NGINX
   - Calico / Canal - CNI used for Kubernetes Pod networking
   - NGINX Application - Mock up GPU management dashboard (built with Claude.ai)
   - ContainerD - Container runtime used by Kubernetes

The following commands were used to deploy this cluster

```bash
#disable swap
swapoff -a

#disable firewall
systemctl stop ufw
systemctl disable ufw

# following commands can be found at kubernetes.io
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install kubectl, kubeadm, kubelet
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable and start kubelet
systemctl enable kubelet && systemctl start kubelet

# Verify status
systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Fri 2026-09-25 19:27:30 UTC; 4h 9min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 6011 (kubelet)
      Tasks: 14 (limit: 4543)
     Memory: 48.0M (peak: 50.4M)
        CPU: 16min 15.538s
     CGroup: /system.slice/kubelet.service

# Deploy Kubernetes cluster
kubeadm init --pod-network-cidr 10.244.0.0/16
```




