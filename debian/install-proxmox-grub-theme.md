# Install Proxmox GRUB Theme

1. Download theme

    ```shell
    sudo apt update
    sudo apt install --yes curl
    ```

    <!-- markdownlint-disable MD013 -->
    ```shell
    curl -SL "https://raw.githubusercontent.com/AdisonCavani/distro-grub-themes/master/themes/proxmox.tar" -o /tmp/proxmox.tar
    sudo mkdir -p /boot/grub/themes/proxmox
    sudo tar -C /boot/grub/themes/proxmox -xf /tmp/proxmox.tar
    rm /tmp/proxmox.tar
    ```
    <!-- markdownlint-enable MD013 -->

2. Edit GRUB config

    <!-- markdownlint-disable MD013 -->
    ```shell
    sudo sed -i 's/#GRUB_GFXMODE=640x480/GRUB_GFXMODE=1280x800/g' /etc/default/grub
    echo -e "\nGRUB_THEME=\"/boot/grub/themes/proxmox/theme.txt\"" | sudo tee -a /etc/default/grub > /dev/null
    ```
    <!-- markdownlint-enable MD013 -->

3. Update GRUB config

    ```shell
    sudo update-grub
    ```
