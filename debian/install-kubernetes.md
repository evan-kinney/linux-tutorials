# Install Kubernetes

1. Disable swap:

    ```shell
    sudo swapoff -a
    ```

    ```shell
    sudo sed -i '/swap/d' /etc/fstab
    ```

    ```shell
    sudo rm -f /swap.img
    ```

2. Load Necessary Kernel Modules:

    Ensure required kernel modules are loaded:

    ```shell
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

    Configure sysctl:

    ```shell
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF
    sudo sysctl --system
    ```

3. Update the apt package index and install packages needed to use the Kubernetes
apt repository:

    ```shell
    sudo apt update
    sudo apt install --yes apt-transport-https \
                            ca-certificates \
                            curl \
                            gnupg
    ```

4. Download the public signing key for the Kubernetes package repositories.
The same signing key is used for all repositories so you can disregard the version
in the URL:

    <!-- markdownlint-disable MD013 -->
    ```shell
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```
    <!-- markdownlint-enable MD013 -->

5. Add the appropriate Kubernetes apt repository. If you want to use Kubernetes
version different than v1.33, replace v1.33 with the desired minor version in
the command below:

    <!-- markdownlint-disable MD013 -->
    ```shell
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
    sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
    ```
    <!-- markdownlint-enable MD013 -->

6. Update apt package index, then install `kubelet`, `kubeadm`, and `kubectl`:

    ```shell
    sudo apt update
    sudo apt install --yes kubelet \
                           kubeadm \
                           kubectl
    ```

7. (Optional) Initialize Kubernetes:

    ```shell
    sudo kubeadm config images pull
    sudo kubeadm init
    ```

8. (Optional) Set Up kubectl for the root user:

    ```shell
    sudo mkdir -p /root/.kube
    sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
    sudo chown $(sudo id -u):$(sudo id -g) /root/.kube/config
    ```

9. (Optional) Set Up kubectl for the current user:

    ```shell
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

10. (Optional) Deploy a pod network

    ```shell
    sudo kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ```

11. (Optional) Enable the Master Node as a Worker Node

    ```shell
    sudo kubectl taint nodes $(hostname | tr '[:upper:]' '[:lower:]') node-role.kubernetes.io/control-plane:NoSchedule-
    ```
