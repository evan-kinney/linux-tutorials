# Install QEMU Guest Agent

1. Install QEMU Guest Agent:

    ```shell
    sudo apt update
    sudo apt install --yes qemu-guest-agent
    ```

2. Start QEMU Guest Agent service:

    ```shell
    sudo systemctl start qemu-guest-agent
    ```

3. Enable QEMU Guest Agent service to run on startup:

    ```shell
    sudo systemctl enable qemu-guest-agent
    ```
