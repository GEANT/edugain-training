# HOWTO Install and Configure SWITCHwayf Centralized Discovery Service


## Table of Contents

1.  [Requirements](#requirements)
    1.  [Hardware](#hardware)
    2.  [Software](#software)
    3.  [Others](#others)
2.  [Notes](#notes)
3.  [Configure the environment](#configure-the-environment)
4.  [Configure APT Mirror](#configure-apt-mirror)
5.  [Install Dependencies](#install-dependencies)
6.  [Install Apache Web Server](#install-apache-web-server)
7.  [Configure Apache Web Server](#configure-apache-web-server)
8.  [Install SWITCHwayf](#install-switchwayf)
9.  [Configure SWITCHwayf as a Centralized Discovery Service](#configure-switchwayf-as-a-centralized-discovery-service)
10. [Configure Apache WAYF VirtualHost](#configure-apache-wayf-virtualhost)
11. [Documentation](#documentation)
12. [Authors](#authors)
13. [Thanks](#thanks)

## Requirements

### Hardware

-   CPU: 2 Core (64 bit)
-   RAM: 4 GB
-   HDD: 10 GB
-   OS: Debian 12 / Ubuntu 22.04

### Software

-   Apache Web Server (*\<= 2.4*)
-   PHP (*7 or 8*)
-   PHP XML Parser extension (Required for parsing XML metadata.)

## Notes

This HOWTO will use `Vim` as text editor:

-   `Esc button + i` means "insert"
-   `Esc button + :w` means "write"
-   `Esc button + :q` means "quit"
-   `Esc button + :wq` means "write & quit"
-   `Esc button + /` means "search text"

[TOC](#table-of-contents)

## Configure the environment

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Be sure that your firewall **is not blocking** the traffic on port **443** and **80** for the SP server.

3.  Set the SP hostname:

    **!!!ATTENTION!!!**: Replace `wayf.example.org` with your SP Full Qualified Domain Name and `<HOSTNAME>` with the SP hostname

    -   ``` text
        echo "<YOUR-SERVER-IP-ADDRESS> wayf.example.org <HOSTNAME>" >> /etc/hosts
        ```

    -   ``` text
        hostnamectl set-hostname <HOSTNAME>

[TOC](#table-of-contents)

## Configure APT Mirror

Debian Mirror List: <https://www.debian.org/mirror/list>

Ubuntu Mirror List: <https://launchpad.net/ubuntu/+archivemirrors>

Example with the Consortium GARR italian mirrors:

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Change the default mirror:

    -   Debian 12 - Deb822 file format:

        ``` text
        bash -c 'cat > /etc/apt/sources.list.d/garr.sources <<EOF
        Types: deb deb-src
        URIs: https://debian.mirror.garr.it/debian/
        Suites: bookworm bookworm-updates bookworm-backports
        Components: main

        Types: deb deb-src
        URIs: https://debian.mirror.garr.it/debian-security/
        Suites: bookworm-security
        Components: main
        EOF'
        ```

    -   Ubuntu:

        ``` text
        bash -c 'cat > /etc/apt/sources.list.d/garr.list <<EOF
        deb https://ubuntu.mirror.garr.it/ubuntu/ jammy main
        deb-src https://ubuntu.mirror.garr.it/ubuntu/ jammy main
        EOF'
        ```

3.  Update packages:

    ``` text
    apt update && apt-get upgrade -y --no-install-recommends
    ```

[TOC](#table-of-contents)

## Install Dependencies

``` text
sudo apt install fail2ban vim wget ca-certificates ntp --no-install-recommends
```

[TOC](#table-of-contents)

## Install Apache Web Server

The Apache HTTP Server will be configured for SSL offloading.

``` text
sudo apt install apache2
```

[TOC](#table-of-contents)

## Configure Apache Web Server

1.  Create the DocumentRoot:

    -   ``` text
        mkdir /var/www/html/$(hostname -f)
        ```

    -   ``` text
        chown -R www-data: /var/www/html/$(hostname -f)
        ```

    -   ``` text
        echo '<h1>It Works!</h1>' > /var/www/html/$(hostname -f)/index.html
        ```

2.  Put SSL credentials in the right place:

    > According to [NSA and NIST](https://www.keylength.com/en/compare/), RSA with 3072 bit-modulus is the minimum to protect up to TOP SECRET over than 2030.

    -   HTTPS Server Certificate (Public Key) inside `/etc/ssl/certs/$(hostname -f).crt`

    -   HTTPS Server Key (Private Key) inside `/etc/ssl/private/$(hostname -f).key`

    -   Add CA Cert into `/etc/ssl/certs`
        -   If you use GEANT TCS:

            -   ``` text
                wget -O /etc/ssl/certs/GEANT_OV_RSA_CA_4.pem https://crt.sh/?d=2475254782
                ```
         
            -   ``` text
                wget -O /etc/ssl/certs/SectigoRSAOrganizationValidationSecureServerCA.crt https://crt.sh/?d=924467857
                ```
         
            -   ``` text
                cat /etc/ssl/certs/SectigoRSAOrganizationValidationSecureServerCA.crt >> /etc/ssl/certs/GEANT_OV_RSA_CA_4.pem
                ```
         
            -   ``` text
                rm /etc/ssl/certs/SectigoRSAOrganizationValidationSecureServerCA.crt
                ```

        -   If you use Let's Encrypt:

            - ``` text
              ln -s /etc/letsencrypt/live/<SERVER_FQDN>/chain.pem /etc/ssl/certs/ACME-CA.pem
              ```

3.  Configure the right privileges for the SSL Certificate and Private Key used by HTTPS:

    -   ``` text
        chmod 400 /etc/ssl/private/$(hostname -f).key
        ```

    -   ``` text
        chmod 644 /etc/ssl/certs/$(hostname -f).crt
        ```

    (`$(hostname -f)` will provide your SP Full Qualified Domain Name)

4.  Verify that SSL certificate file matches the CA certificate file (`/etc/ssl/certs/GEANT_OV_RSA_CA_4.pem` or `/etc/ssl/certs/ACME-CA.pem`) with:

    - ``` text
      openssl verify --CAfile <YOUR CA FILE> /etc/ssl/certs/$(hostname -f).crt
      ```

    and make sure you get an `OK` as an outcome.

5.  Enable the required Apache2 modules and the virtual hosts:

    -   ``` text
        a2enmod ssl headers alias include negotiation
        ```

    -   ``` text
        a2dissite 000-default.conf default-ssl
        ```

    -   ``` text
        systemctl restart apache2.service
        ```

[TOC](#table-of-contents)

## Install SWITCHwayf

1. Become ROOT:

    ``` text
    sudo su -
    ```

2. Install dependencies:

   * ``` text
     apt install php php-xml git --no-install-recommends
     ```

3. Install SWITCHwayf:

   * ``` text
     git clone https://gitlab.switch.ch/aai/SWITCHwayf.git /opt/SWITCHwayf
     ```

[TOC](#table-of-contents)

## Configure SWITCHwayf as a Centralized Discovery Service

1. Become ROOT:

    ``` text
    sudo su -
    ```

2. Create the needed configuration files and directories:

   * ``` text
     mkdir /opt/SWITCHwayf/metadata
     ```

   * ``` text
     cd /opt/SWITCHwayf/etc
     ```

   * ``` text
     cp config.dist.php config.php
     ```

   * ``` text
     cp IDProvider.conf.dist.php IDProvider.conf.php
     ```

3. Complete the configuration file templates (read comments to understand each field configured):

   * ``` text
     vim /opt/SWITCHwayf/etc/config.php
     ```

     ``` text
     $commonDomain = '.example.org';
     ```

     ``` text
     $cookieNamePrefix = '_example-wayf';
     ```

     ``` text
     $useSAML2Metadata = true;
     ```

     ``` text
     $includeLocalConfEntries = false;
     ```

     ``` text
     $enableDSReturnParamCheck = true;
     ```

     ``` text
     $useACURLsForReturnParamCheck = true;
     ```

     ``` text
     $metadataFile = '/opt/SWITCHwayf/metadata/metadata.xml';
     ```

     (*Replace `.example.org` with your WAYF Domain Name and `_example-wayf` with your preferred value*)

4. Create the SWITCHwayf cron job:

   * ``` text
     touch /etc/cron.hourly/wayf-update
     ```

   * ``` text
     chmod +x /etc/cron.hourly/wayf-update
     ```

   * ``` text
     bash -c 'cat > /etc/cron.hourly/wayf-update <<EOF
     #!/bin/bash
  
     wget <FEDERATION-METADATA-URL> -O /opt/SWITCHwayf/metadata/metadata.xml >> /var/log/apache2/wayf.log
  
     php /opt/SWITCHwayf/bin/update-metadata.php --metadata-file /opt/SWITCHwayf/metadata/metadata.xml --metadata-idp-file /opt/SWITCHwayf/etc/IDProvider.metadata.php --metadata-sp-file /opt/SWITCHwayf/etc/SProvider.metadata.php --verbose >> /var/log/apache2/wayf.log
     EOF'
     ```

[TOC](#table-of-contents)

## Configure Apache WAYF VirtualHost

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Create the Virtualhost file:

    ``` text
    wget https://raw.githubusercontent.com/GEANT/edugain-training/main/UbuntuNet-Training-202401/config-files/shibboleth/SP3/apache/sp.example.org.conf -O /etc/apache2/sites-available/$(hostname -f).conf
    ```

3.  Edit the Virtualhost file (**PLEASE PAY ATTENTION! You need to edit this file and customize it, check the initial comment of the file**):

    ``` text
    vim /etc/apache2/sites-available/$(hostname -f).conf
    ```

    and enrich the Apache VirtualHost configuration with the following statement:

    ``` text
    DocumentRoot /opt/SWITCHwayf/www     

     <Directory /opt/SWITCHwayf/www>
         Options Indexes MultiViews
         AllowOverride None
         #Order allow,deny
         #Allow from all
         Require all granted
     
       <Files WAYF>
           SetHandler php-script
           # SetHandler php7-script
           AcceptPathInfo On
       </Files>
     </Directory>
    ```

4. Enable the Apache2 WAYF Virtualhosts configured:

    -   ``` text
        a2ensite $(hostname -f).conf
        ```

    -   ``` text
        systemctl reload apache2.service
        ```

5.  Check that WAYF web server works on:

    ``` text
    https://wayf.example.org/WAYF
    ```

    (*Replace `wayf.example.org` with your WAYF Full Qualified Domain Name*)

6.  Verify the strength of your WAYF's machine on [SSLLabs](https://www.ssllabs.com/ssltest/analyze.html).

[TOC](#table-of-contents)

## Documentation

- [SWITCHwayf Repository](https://gitlab.switch.ch/aai/SWITCHwayf)

[TOC](#table-of-contents)

## Authors

### Original Author

 * Marco Malavolti (marco.malavolti@garr.it)

## Thanks

 * [SWITCHaai](https://help.switch.ch/aai/)

[TOC](#table-of-contents)
