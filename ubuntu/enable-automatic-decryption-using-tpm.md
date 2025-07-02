# Enable automatic decryption using TPM

## Step 1: Install required packages

```shell
sudo apt update
sudo apt install --yes clevis \
                       clevis-tpm2 \
                       clevis-luks \
                       clevis-initramfs \
                       initramfs-tools \
                       tss2 \
                       tpm2-tools
```

## Step 2: Bind LUKS volume to TPM

Replace `scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-part3` with your disk partition:

<!-- markdownlint-disable MD013 -->
```shell
sudo clevis luks bind \
  -d /dev/disk/by-uuid/$(blkid -s UUID -o value /dev/disk/by-id/scsi-0QEMU_QEMU_HARDDISK_drive-scsi0-part3) \
  tpm2 '{"pcr_bank":"sha256"}'
```
<!-- markdownlint-enable MD013 -->

## Step 3: Update initramfs

```shell
sudo update-initramfs -u -k all
```
