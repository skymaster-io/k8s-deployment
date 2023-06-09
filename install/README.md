# Installing Kubernetes on Rocky Linux with Containerd

This guide provides step-by-step instructions for installing Kubernetes with Containerd on Rocky Linux.

## Prerequisites

- Host machines running Rocky Linux that meet the minimum hardware requirements for Kubernetes.

## Installation Steps

1. **Set up the Host Machines**

   - Prepare your host machines running Rocky Linux.
   - Ensure the machines meet the minimum hardware requirements for running Kubernetes.

2. **Install Containerd**

   - Install Containerd on each host machine by running the following commands:

     ```shell
     sudo yum install -y yum-utils
     sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
     sudo dnf update
     sudo dnf install -y containerd iproute-tc
     ```

3. **Configure Containerd**

   - Create the `/etc/containerd` directory:
     ```
     sudo mkdir /etc/containerd
     ```

   - Create the `/etc/containerd/config.toml` configuration file:
     ```
     sudo nano /etc/containerd/config.toml
     ```

   -Rreplace the following configuration to the file:
     ```toml
     [plugins."io.containerd.grpc.v1.cri"]
       sandbox_image = "docker.io/library/busybox:latest"
       [plugins."io.containerd.grpc.v1.cri".containerd]
         snapshotter = "overlayfs"
         [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
           runtime_type = "io.containerd.runtime.v1.linux"
     ```

   - Save and close the file.

4. **Install Kubernetes Components**

   - Install the necessary packages by running the following commands:
     
   ```shell
   cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
   [kubernetes]
   name=Kubernetes
   baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
   enabled=1
   gpgcheck=1
   gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
   exclude=kubelet kubeadm kubectl
   EOF
   ```

# Set SELinux in permissive mode (effectively disabling it)
```shell
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

5. **Enable and Start Services**

   - Enable and start the `containerd` service:
     ```
     sudo systemctl enable --now containerd
     ```

   - Enable and start the `kubelet` service:
     ```
     sudo systemctl enable kubelet
     ```

6. **Initialize the Kubernetes Cluster**

   - On the machine that will be the master node, initialize the cluster by running:
    ```
    echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
    echo 'net.bridge.bridge-nf-call-iptables = 1' >>/etc/sysctl.conf
    modprobe br_netfilter
    echo br_netfilter >> /etc/modules-load.d/br_netfilter.conf
    sysctl -p
    systemctl disable firewalld
    systemctl stop firewalld    
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16
    ```

   - Once the command completes, it will output a `kubeadm join` command. Save this command as you will need it to join worker nodes to the cluster.

7. **Set Up the kubeconfig File**

   - Run the following commands to set up the kubeconfig file:
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

8. **Install a Network Plugin**

?? https://docs.tigera.io/calico/latest/getting-started/kubernetes/hardway/install-cni-plugin


   - Install a network plugin to enable pod networking. For example, you can use Calico by running the following command:
     ```
     kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
     ```

9. **Join Worker Nodes**

   - On each worker node, run the `kubeadm join` command you saved earlier.

10. **Verify Cluster**

    - On the master node, run the following command to check the status of the cluster and ensure all nodes are ready:
      ```
      kubectl get nodes
      ```

Congratulations! You have now installed Kubernetes with Containerd on Rocky Linux. Your cluster is ready to deploy and manage containerized applications.

