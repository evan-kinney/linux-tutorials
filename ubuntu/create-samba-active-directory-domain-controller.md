# Create Samba Active Directory Domain Controller

## Step 1: Set the Server Hostname

Replace `HOSTNAME` with the desired hostname:

```shell
sudo hostname HOSTNAME
sudo hostnamectl set-hostname $(hostname)
sudo nano /etc/hosts
```

Replace `192.168.1.2` with the IP address of the server and `FQDN` with the fully
qualified domain of the server:

```text
Add the line:
192.168.1.2      FQDN HOSTNAME
```

## Step 2: Disable the DNS Resolver

1. Disable `systemd-resolved`:

   ```shell
   sudo systemctl disable --now systemd-resolved
   ```

1. Create `resolv.conf`:

   ```shell
   sudo unlink /etc/resolv.conf
   ```

   ```shell
   sudo nano /etc/resolv.conf
   ```

   Replace `192.168.1.2` with the IP address of the server and `ad.example.com`
   with your domain:

   ```text
   Add the lines:
   nameserver 192.168.1.2
   nameserver 1.1.1.1
   nameserver 1.0.0.1
   search ad.example.com
   ```

   ```shell
   sudo chattr +i /etc/resolv.conf
   ```

## Step 3: Install Samba

1. Install required packages:

   ```shell
   sudo apt update
   sudo apt install --yes acl \
                          attr \
                          samba \
                          samba-dsdb-modules \
                          samba-vfs-modules \
                          smbclient \
                          winbind \
                          libpam-winbind \
                          libnss-winbind \
                          libpam-krb5 \
                          krb5-config \
                          krb5-user \
                          dnsutils \
                          chrony \
                          net-tools \
                          samba-ad-dc
   ```

   Replace `FQDN` with the fully qualified domain name used above and `AD.EXAMPLE.COM`
   with your domain:

   ```shell
   # Default Kerberos version 5 realm:
   AD.EXAMPLE.COM
   ```

   ```shell
   # Kerberos servers for your realm:
   FQDN
   ```

   ```shell
   # Administrative server for your Kerberos realm:
   FQDN
   ```

1. Disable Samba services:

   ```shell
   sudo systemctl disable --now smbd \
                                nmbd \
                                winbind
   ```

1. Activate `samba-ad-dc` service:

   ```shell
   sudo systemctl unmask samba-ad-dc
   sudo systemctl enable samba-ad-dc
   ```

## Step 4: Configuring Samba Active Directory

1. Backup `smb.conf`:

   ```shell
   sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.orig
   ```

1. Provision domain:

   ```shell
   sudo samba-tool domain provision
   ```

   Replace `FQDN` with the fully qualified domain name used above and
   `AD.EXAMPLE.COM` with your domain:

   ```shell
   # Realm:
   AD.EXAMPLE.COM
   ```

   ```shell
   # Domain:
   AD
   ```

   ```shell
   # Server Role:
   dc
   ```

   ```shell
   # DNS backend:
   SAMBA_INTERNAL
   ```

   ```shell
   # DNS forward IP address:
   1.1.1.1
   ```

1. Backup and replace the Kerberos Config:

   ```shell
   sudo mv /etc/krb5.conf /etc/krb5.conf.orig
   sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
   ```

1. (Optional) Set custom TLS certificate:

   Copy your PEM certificates and key to:

      - `/var/lib/samba/private/tls/cert.pem`
      - `/var/lib/samba/private/tls/ca.pem`
      - `/var/lib/samba/private/tls/key.pem`

1. Start `samba-ad-dc` service:

   ```shell
   sudo systemctl start samba-ad-dc
   sudo systemctl status samba-ad-dc
   ```

## Step 5: Set up Time Synchronization

1. Set permissions:

   Allow group `_chrony` to read the directory `ntp_signd`

   ```shell
   sudo chown root:_chrony /var/lib/samba/ntp_signd/
   ```

   Change the permission of the directory `ntp_signd`

   ```shell
   sudo chmod 750 /var/lib/samba/ntp_signd/
   ```

1. Update `chrony.conf`:

   ```shell
   sudo nano /etc/chrony/chrony.conf
   ```

   Replace `192.168.1.2` with the IP address of the server and `192.168.1.0/24`
   with your subnet:

   ```txt
   Add the lines:
   bindcmdaddress 192.168.1.2
   allow 192.168.1.0/24
   ntpsigndsocket /var/lib/samba/ntp_signd
   ```

1. Restart `chronyd` service:

   ```shell
   sudo systemctl restart chronyd
   sudo systemctl status chronyd
   ```

## Step 6: Verifying Samba Active Directory

1. Verify the domain:

   Replace `ad.example.com` with your domain:

   ```shell
   host -t A ad.example.com
   ```

1. Verify the domain controller:

   Replace `FQDN` with the fully qualified domain name used above:

   ```shell
   host -t A $(hostname -f)
   ```

1. Verify Kerberos service:

   Replace `ad.example.com` with your domain:

   ```shell
   host -t SRV _kerberos._udp.ad.example.com
   ```

1. Verify LDAP service:

   Replace `ad.example.com` with your domain:

   ```shell
   host -t SRV _ldap._tcp.ad.example.com
   ```

## Step 7: Allow domain users to login locally

1. Configure NSS:

   ```shell
   sudo nano /etc/nsswitch.conf
   ```

   ```text
   # Edit the lines:
   passwd:         files systemd winbind
   group:          files systemd winbind
   ```

1. Enable automatic home directory creation:

   ```shell
   sudo pam-auth-update --enable mkhomedir
   ```

1. Setup `sudo` for domain:

    Replace `ad.example.com` with your domain:

    ```shell
    sudo nano /etc/sudoers.d/domain
    ```

    Replace `AD` with your domain:

    ```text
    %AD\\Administrators ALL=(ALL) ALL
    %AD\\Domain\ Admins ALL=(ALL) ALL
    %AD\\Enterprise\ Admins ALL=(ALL) ALL
    ```
