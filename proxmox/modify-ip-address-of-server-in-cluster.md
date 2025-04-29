# Modify IP Address of Server in Cluster

## Step 1: Change network address of server

1. Update `interfaces`:

    ```shell
    nano /etc/network/interfaces
    ```

1. Update `hosts`:

    ```shell
    nano /etc/hosts
    ```

## Step 2: Update network address in cluster

1. Stop `pve-cluster` service:

    ```shell
    systemctl stop pve-cluster
    killall pmxcfs
    ```

2. Force local access to `corosync.conf`:

    ```shell
    pmxcfs -l
    ```

3. Modify `corosync.conf`:

    ```shell
    nano /etc/pve/corosync.conf
    ```

4. Restart `pve-cluster` service:

    ```shell
    killall pmxcfs
    systemctl start pve-cluster
    ```

5. Reread `corosync` configuration:

    ```shell
    corosync-cfgtool -R
    ```

6. Print `corosync` configuration:

    ```shell
    corosync-cfgtool -s
    ```

7. Restart:

    ```shell
    reboot
    ```
