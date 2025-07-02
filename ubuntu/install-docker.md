# Install docker

1. Set up Docker's `apt` repository

    Add Docker's official GPG key:

    ```shell
    sudo apt update
    sudo apt install ca-certificates curl
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

    Add the repository to Apt sources:

    <!-- markdownlint-disable MD013 -->
    ```shell
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt update
    ```
    <!-- markdownlint-enable MD013 -->

2. Install the Docker packages

    ```shell
    sudo apt install --yes docker-ce \
                           docker-ce-cli \
                           containerd.io \
                           docker-buildx-plugin \
                           docker-compose-plugin
    ```

3. (Optional) Initialize swarm

    ```shell
    sudo docker swarm init
    ```

4. (Optional) Configure containerd

    - Generate the default configuration file for containerd:

        ```shell
        sudo mkdir -p /etc/containerd
        containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
        ```

    - Ensure containerd uses systemd as the cgroup driver:

        ```shell
        sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
        ```

    - Restart containerd:

        ```shell
        sudo systemctl restart containerd
        sudo systemctl enable containerd
        ```

5. (Optional) Create `br0` network

    Replace `192.168.1.0/24` and `192.168.1.1` with your network

    ```shell
    sudo docker network create \
      --driver macvlan \
      --subnet=192.168.1.0/24 \
      --gateway=192.168.1.1 \
      --opt parent=eth0 \
      br0
    ```
