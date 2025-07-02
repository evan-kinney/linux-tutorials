# Create Certificate Authority

1. Install OpenSSL:

    ```shell
    sudo apt install --yes openssl
    ```

1. Create Certificate Authority directories:

    ```shell
    sudo mkdir -p /etc/certificates/{certs,crl,newcerts,private}
    sudo chmod -R 700 /etc/certificates/private
    sudo touch /etc/certificates/index.txt
    echo 1000 | sudo tee /etc/certificates/serial >/dev/null
    echo 1000 | sudo tee /etc/certificates/crlnumber >/dev/null
    ```

1. Create new `openssl.cnf` for generating Certificate Authority:

    Replace `example.com` with your domain:

    ```shell
    sudo nano /etc/certificates/private/CA.example.com.openssl.cnf
    ```

    Replace `example.com` with your domain and update `[ req_distinguished_name ]`
    to match your domain:

    ```ini
    [ ca ]
    default_ca = CA_default

    [ CA_default ]
    dir             = /etc/certificates
    certs           = $dir/certs
    crl_dir         = $dir/crl
    database        = $dir/index.txt
    new_certs_dir   = $dir/newcerts
    serial          = $dir/serial
    crlnumber       = $dir/crlnumber
    crl             = $dir/crl/CA.example.com.crl.pem
    private_key     = $dir/private/CA.example.com.key
    certificate     = $dir/certs/CA.example.com.crt
    default_days    = 7300
    default_md      = sha256
    preserve        = no
    policy          = policy_strict
    email_in_dn     = no
    name_opt        = ca_default
    cert_opt        = ca_default

    [ policy_strict ]
    countryName             = match
    stateOrProvinceName     = match
    localityName            = match
    organizationName        = match
    organizationalUnitName  = match
    commonName              = supplied
    emailAddress            = optional

    [ req ]
    default_bits            = 4096
    default_md              = sha256
    distinguished_name      = req_distinguished_name
    x509_extensions         = v3_ca

    [ req_distinguished_name ]
    countryName                      = Country Name (2 letter code)
    countryName_default              = 
    stateOrProvinceName              = State or Province Name
    stateOrProvinceName_default      = 
    localityName                     = Locality Name
    localityName_default             = 
    organizationName                 = Organization Name
    organizationName_default         = 
    organizationalUnitName           = Organization Unit Name
    commonName                       = Common Name
    commonName_max                   = 64
    emailAddress                     = Email Address
    emailAddress_max                 = 64

    [ v3_ca ]
    basicConstraints = critical, CA:true
    keyUsage = critical, keyCertSign, cRLSign
    subjectKeyIdentifier = hash
    ```

    ```shell
    sudo chattr +i /etc/certificates/private/CA.example.com.openssl.cnf
    ```

1. Generate Certificate Authority private key password:

    Replace `example.com` with your domain:

    <!-- markdownlint-disable MD013 -->
    ```shell
    openssl rand -base64 512 | \
      tr -dc 'A-Za-z0-9' | \
      head -c 256 | sudo tee /etc/certificates/private/CA.example.com.password.txt >/dev/null
    sudo chattr +i /etc/certificates/private/CA.example.com.password.txt
    ```
    <!-- markdownlint-enable MD013 -->

1. Generate Certificate Authority private key:

    Replace `example.com` with your domain:

    ```shell
    sudo openssl genpkey \
      -algorithm RSA \
      -out /etc/certificates/private/CA.example.com.key \
      -aes256 \
      -pkeyopt rsa_keygen_bits:4096 \
      -pass file:/etc/certificates/private/CA.example.com.password.txt
    sudo chattr +i /etc/certificates/private/CA.example.com.key
    ```

1. Generate Certificate Authority Certificate Signing Request:

    Replace `example.com` with your domain:

    ```shell
    sudo openssl req \
      -new \
      -key /etc/certificates/private/CA.example.com.key \
      -passin file:/etc/certificates/private/CA.example.com.password.txt \
      -out /etc/certificates/private/CA.example.com.csr \
      -config /etc/certificates/private/CA.example.com.openssl.cnf
    sudo chattr +i /etc/certificates/private/CA.example.com.csr
    ```

1. Generate Certificate Authority Certificate:

    Replace `example.com` with your domain:

    ```shell
    openssl x509 \
      -req \
      -in /etc/certificates/private/CA.example.com.csr \
      -key /etc/certificates/private/CA.example.com.key \
      -passin file:/etc/certificates/private/CA.example.com.password.txt \
      -out /etc/certificates/private/CA.example.com.crt \
      -days 7300 \
      -sha256 \
      -extensions v3_ca \
      -extfile /etc/certificates/private/CA.example.com.openssl.cnf
    sudo chattr +i /etc/certificates/private/CA.example.com.crt
    ```

1. Create new `openssl.cnf` for generating intermediate Certificate Authority:

    Replace `example.com` with your domain:

    ```shell
    sudo nano /etc/certificates/private/Intermediate-CA.example.com.openssl.cnf
    ```

    Replace `example.com` with your domain and update `[ req_distinguished_name ]`
    to match your domain:

    ```ini
    [ ca ]
    default_ca = CA_default

    [ CA_default ]
    dir             = /etc/certificates
    certs           = $dir/certs
    crl_dir         = $dir/crl
    database        = $dir/index.txt
    new_certs_dir   = $dir/newcerts
    serial          = $dir/serial
    crlnumber       = $dir/crlnumber
    crl             = $dir/crl/Intermediate-CA.example.com.crl.pem
    private_key     = $dir/private/Intermediate-CA.example.com.key
    certificate     = $dir/certs/Intermediate-CA.example.com.crt
    default_days    = 3650
    default_md      = sha256
    preserve        = no
    policy          = policy_loose

    [ policy_loose ]
    countryName             = optional
    stateOrProvinceName     = optional
    localityName            = optional
    organizationName        = optional
    organizationalUnitName  = optional
    commonName              = supplied
    emailAddress            = optional

    [ req ]
    default_bits            = 4096
    default_md              = sha256
    distinguished_name      = req_distinguished_name
    attributes              = req_attributes
    x509_extensions         = v3_intermediate_ca

    [ req_distinguished_name ]
    countryName                      = Country Name (2 letter code)
    countryName_default              = 
    stateOrProvinceName              = State or Province Name
    stateOrProvinceName_default      = 
    localityName                     = Locality Name
    localityName_default             = 
    organizationName                 = Organization Name
    organizationName_default         = 
    organizationalUnitName           = Organization Unit Name
    commonName                       = Common Name
    commonName_max                   = 64
    emailAddress                     = Email Address
    emailAddress_max                 = 64

    [ req_attributes ]
    challengePassword                = A challenge password
    challengePassword_min            = 0
    challengePassword_max            = 64

    [ v3_intermediate_ca ]
    basicConstraints = critical, CA:true, pathlen:0
    keyUsage = critical, digitalSignature, keyCertSign, cRLSign
    extendedKeyUsage = serverAuth, clientAuth
    subjectKeyIdentifier = hash
    authorityKeyIdentifier = keyid:always, issuer
    ```

    ```shell
    sudo chattr +i /etc/certificates/private/Intermediate-CA.example.com.openssl.cnf
    ```

1. Generate Intermediate Certificate Authority private key password:

    Replace `example.com` with your domain:

    <!-- markdownlint-disable MD013 -->
    ```shell
    openssl rand -base64 512 | \
      tr -dc 'A-Za-z0-9' | \
      head -c 256 | sudo tee /etc/certificates/private/Intermediate-CA.example.com.password.txt >/dev/null
    sudo chattr +i /etc/certificates/private/Intermediate-CA.example.com.password.txt
    ```
    <!-- markdownlint-enable MD013 -->

1. Generate Intermediate Certificate Authority private key:

    Replace `example.com` with your domain:

    ```shell
    sudo openssl genpkey \
      -algorithm RSA \
      -out /etc/certificates/private/Intermediate-CA.example.com.key \
      -aes256 \
      -pkeyopt rsa_keygen_bits:4096 \
      -pass file:/etc/certificates/private/Intermediate-CA.example.com.password.txt
    sudo chattr +i /etc/certificates/private/Intermediate-CA.example.com.key
    ```

1. Generate Intermediate Certificate Authority Certificate Signing Request:

    Replace `example.com` with your domain:

    <!-- markdownlint-disable MD013 -->
    ```shell
    sudo openssl req \
      -new \
      -key /etc/certificates/private/Intermediate-CA.example.com.key \
      -passin file:/etc/certificates/private/Intermediate-CA.example.com.password.txt \
      -out /etc/certificates/private/Intermediate-CA.example.com.csr \
      -config /etc/certificates/private/Intermediate-CA.example.com.openssl.cnf
    sudo chattr +i /etc/certificates/private/Intermediate-CA.example.com.csr
    ```
    <!-- markdownlint-enable MD013 -->

1. Generate Intermediate Certificate Authority Certificate:

    Replace `example.com` with your domain:

    ```shell
    sudo openssl x509 \
      -req \
      -in /etc/certificates/private/Intermediate-CA.example.com.csr \
      -CA /etc/certificates/private/CA.example.com.crt \
      -CAkey /etc/certificates/private/CA.example.com.key \
      -passin file:/etc/certificates/private/CA.example.com.password.txt \
      -CAcreateserial \
      -out /etc/certificates/private/Intermediate-CA.example.com.crt \
      -days 3650 \
      -sha256 \
      -extensions v3_intermediate_ca \
      -extfile /etc/certificates/private/Intermediate-CA.example.com.openssl.cnf
    sudo chattr +i /etc/certificates/private/Intermediate-CA.example.com.crt
    ```
