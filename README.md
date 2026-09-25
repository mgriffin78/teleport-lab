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
disable swap and turn off Ubuntu FW
```bash
#disable swap
swapoff -a

#disable firewall
systemctl stop ufw
systemctl disable ufw
```

