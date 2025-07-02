# Install Helm

1. Update the apt package index and install packages needed to use the Helm apt repository:

    ```shell
    sudo apt update
    sudo apt install --yes apt-transport-https \
                           curl
    ```

2. Download the public signing key for the Helm package repositories.

    ```shell
    curl https://baltocdn.com/helm/signing.asc | sudo gpg --dearmor -o /usr/share/keyrings/helm.gpg
    sudo chmod 644 /usr/share/keyrings/helm.gpg
    ```

3. Add the appropriate Helm apt repository

    <!-- markdownlint-disable MD013 -->
    ```shell
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    ```
    <!-- markdownlint-enable MD013 -->

4. Update apt package index, then install `helm`:

    ```shell
    sudo apt update
    sudo apt install --yes helm
    ```
