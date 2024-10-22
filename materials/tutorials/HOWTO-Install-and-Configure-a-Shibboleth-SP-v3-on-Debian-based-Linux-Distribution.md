# HOWTO Install and Configure a Shibboleth SP v3 on Debian based Linux Distribution

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
8.  [Configure Apache SP VirtualHost](#configure-apache-sp-virtualhost)
9.  [Install Shibboleth Service Provider](#install-shibboleth-service-provider)
10. [Configure Shibboleth Service Provider](#configure-shibboleth-service-provider) 
11. [Configure an example federated resource "secure"](#configure-an-example-federated-resource-secure)
12. [Enable Attribute Support on Shibboleth Service Provider](#enable-attribute-support-on-shibboleth-service-provider)
13. [Connect a Service Provider directly to an Identity Provider](#connect-a-service-provider-directly-to-an-identity-provider)
14. [Test](#test)
15. [Enable Attribute Checker Support on Shibboleth Service Provider](#enable-attribute-checker-support-on-shibboleth-service-provider)
16. [Increase startup timeout](#increase-startup-timeout)
17. [OPTIONAL - Maintain 'shibd' working](#optional---maintain-shibd-working)
18. [Utility](#utility)
19. [Authors](#authors)
20. [Thanks](#thanks)

## Requirements

### Hardware

-   CPU: 2 Core (64 bit)
-   RAM: 2 GB (with MDQ service), 4GB (without MDQ service)
-   HDD: 10 GB
-   OS: Debian 12 / Ubuntu 22.04

### Software

-   Apache Web Server (*\<= 2.4*)
-   OpenSSL (*\<= 3.0.2*)
-   Shibboleth Service Provider (*\<= 3.4.1*)
-   PHP (*\<= 8.1*)

### Others

-   SSL Credentials: HTTPS Certificate & Private Key
-   Logo:
    -   size: 64px by 350px wide and 64px by 146px high
    -   format: PNG
    -   style: with a transparent background

[TOC](#table-of-contents)

## Notes

This HOWTO uses `example.org` and `sp.example.org` as example values.

Please remember to **replace all occurencences** of:

-   the `example.org` value with the SP domain name
-   the `sp.example.org` value with the Full Qualified Domain Name of the Service Provider.

[TOC](#table-of-contents)

## Configure the environment

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Be sure that your firewall **is not blocking** the traffic on port **443** and **80** for the SP server.

3.  Set the SP hostname:

    **!!!ATTENTION!!!**: Replace `sp.example.org` with your SP Full Qualified Domain Name and `<HOSTNAME>` with the SP hostname

    -   ``` text
        echo "<YOUR-SERVER-IP-ADDRESS> sp.example.org <HOSTNAME>" >> /etc/hosts
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
sudo apt install fail2ban vim wget ca-certificates openssl ntp --no-install-recommends
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

4.  Configure the right privileges for the SSL Certificate and Private Key used by HTTPS:

    -   ``` text
        chmod 400 /etc/ssl/private/$(hostname -f).key
        ```

    -   ``` text
        chmod 644 /etc/ssl/certs/$(hostname -f).crt
        ```

    (`$(hostname -f)` will provide your SP Full Qualified Domain Name)

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

## Configure Apache SP VirtualHost

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Create the Virtualhost file (**PLEASE PAY ATTENTION! you need to edit this file and customize it, check the initial comment of the file**):

    ``` text
    wget https://raw.githubusercontent.com/GEANT/edugain-training/main/UbuntuNet-Training-202401/config-files/shibboleth/SP3/apache/sp.example.org.conf -O /etc/apache2/sites-available/$(hostname -f).conf
    ```

3.  Enable the Apache2 SP Virtualhosts created:

    -   ``` text
        a2ensite $(hostname -f).conf
        ```

    -   ``` text
        systemctl reload apache2.service
        ```

4.  Check that SP web server works on:

    ``` text
    https://sp.example.org
    ```

6.  Verify the strength of your SP's machine on [SSLLabs](https://www.ssllabs.com/ssltest/analyze.html).

[TOC](#table-of-contents)

## Install Shibboleth Service Provider

1. Become ROOT:

    ``` text
    sudo su -
    ```

3. Install Shibboleth SP:
   * ``` text
     apt install libapache2-mod-shib --no-install-recommends
     ```

   From this point the location of the SP directory is: `/etc/shibboleth`

[TOC](#table-of-contents)

## Configure Shibboleth Service Provider

1. Become ROOT: 

    ``` text
    sudo su -
    ```

2. Change the SP entityID and technical contact email address:
   
   - ``` text
     sed -i "s/sp.example.org/$(hostname -f)/" /etc/shibboleth/shibboleth2.xml
     ```
   - ``` text
     sed -i "s/root@localhost/<TECH-CONTACT-EMAIL-ADDRESS-HERE>/" /etc/shibboleth/shibboleth2.xml
     ```
   - ``` text
     sed -i 's/handlerSSL="false"/handlerSSL="true"/' /etc/shibboleth/shibboleth2.xml
     ```
   - ``` text
     sed -i 's/cookieProps="http"/cookieProps="https"/' /etc/shibboleth/shibboleth2.xml
     ```
   - ``` text
     sed -i 's/cookieProps="https">/cookieProps="https" redirectLimit="exact">/' /etc/shibboleth/shibboleth2.xml
     ```

4. Create SP metadata Signing and Encryption credentials:

   * Ubuntu:

     - ``` text
       cd /etc/shibboleth
       ```
     - ``` text
       shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f
       ```
     - ``` text
       shib-keygen -u _shibd -g _shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f
       ```
     - ``` text
       /usr/sbin/shibd -t
       ```
     - ``` text
       systemctl restart shibd.service
       ```
     - ``` text
       systemctl restart apache2.service
       ```

   * Debian

     - ``` text
       cd /etc/shibboleth
       ```
     - ``` text
       ./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-signing -f
       ```
     - ``` text 
       ./keygen.sh -u shibd -g shibd -h $(hostname -f) -y 30 -e https://$(hostname -f)/shibboleth -n sp-encrypt -f
       ```
     - ``` text
       LD_LIBRARY_PATH=/opt/shibboleth/lib64 /usr/sbin/shibd -t
       ```
     - ``` text
       systemctl restart shibd.service
       ```
     - ``` text
       systemctl restart apache2.service
       ```

5. Enable Shibboleth Apache2 configuration:

   * ``` text
     a2enmod shib
     ```
   * ``` text
     systemctl reload apache2.service
     ```

7. Now you are able to reach your Shibboleth SP Metadata on:
   * ht<span>tps://</span>sp.example.org/Shibboleth.sso/Metadata

     (*Replace `sp.example.org` with your SP Full Qualified Domain Name*)

[TOC](#table-of-contents)

## Configure an example federated resource "secure"

1. Create the Apache2 configuration for the application:

    * ``` text
      sudo su -
      ```

   * ``` text
     vim /etc/apache2/conf-available/secure.conf
     ```
  
     ``` text
     RedirectMatch    ^/$  /secure

     <Location /secure>
       Authtype shibboleth
       ShibRequireSession On
       require valid-user
     </Location>
     ```

   * ``` text
     a2enconf secure
     ```

3. Create the "`secure`" application into the DocumentRoot:

   * ``` text
     mkdir -p /var/www/html/$(hostname -f)/secure
     ```

   * ``` text
     wget https://raw.githubusercontent.com/GEANT/edugain-training/main/UbuntuNet-Training-202401/config-files/shibboleth/SP3/secure/index.php.txt -O /var/www/html/$(hostname -f)/secure/index.php
     ```

4. Install needed packages and restart Apache2:

   * ``` text
     apt install libapache2-mod-php php
     ```
   * ``` text
     systemctl restart apache2.service
     ```

[TOC](#table-of-contents)

## Enable Attribute Support on Shibboleth Service Provider

> The Attribute Map file is used by the Service Provider to recognize and support new attributes released by an Identity Provider

Enable attribute support by removing comment from the related content into `/etc/shibboleth/attribute-map.xml` than restart `shibd` service:

* ``` text
  sudo systemctl restart shibd.service
  ```

[TOC](#table-of-contents)

## Connect a Service Provider directly to an Identity Provider

> Follow these steps **IF** you need to connect one SP to a specific IdP. It is useful for test purposes.

1. Edit `shibboleth2.xml` opportunely:

   * ``` text
     vim /etc/shibboleth/shibboleth2.xml
     ```

     ```bash

     <!-- If it is needed to manage the authentication on several IdPs
          install and configure the Shibboleth Embedded Discovery Service
          by following the "HOWTO Install and Configure a Shibboleth Embedded Discovery Service"
     -->
     <SSO entityID="https://idp.example.org/idp/shibboleth">
        SAML2
     </SSO>
     <!-- ... other things ... -->
     <MetadataProvider type="XML" validate="true"
                       url="https://idp.example.org/idp/shibboleth"
                       backingFilePath="idp-metadata.xml" maxRefreshDelay="7200" />
     ```
 
     (*Replace `entityID` with the IdP entityID and `url` with an URL where it can be downloaded its metadata*)
     
     (`idp-metadata.xml` will be saved into `/var/cache/shibboleth`)
 
3. Restart `shibd` and `Apache2` daemon:
   * `sudo systemctl restart shibd`
   * `sudo systemctl restart apache2`

4. Jump to [Test](#test)

[TOC](#table-of-contents)

## Test

Open the `https://sp.example.org/secure` application into your web browser

(*Replace `sp.example.org` with your SP Full Qualified Domain Name*)

[TOC](#table-of-contents)

## Enable Attribute Checker Support on Shibboleth Service Provider

1. Add a sessionHook for attribute checker: `sessionHook="/Shibboleth.sso/AttrChecker"` and the `metadataAttributePrefix="Meta-"` to `ApplicationDefaults`:
   * ``` text
     vim /etc/shibboleth/shibboleth2.xml`
     ```

     ``` text
     <ApplicationDefaults entityID="https://sp.example.org/shibboleth"
                          REMOTE_USER="eppn subject-id pairwise-id persistent-id"
                          cipherSuites="DEFAULT:!EXP:!LOW:!aNULL:!eNULL:!DES:!IDEA:!SEED:!RC4:!3DES:!kRSA:!SSLv2:!SSLv3:!TLSv1:!TLSv1.1"
                          sessionHook="/Shibboleth.sso/AttrChecker"
                          metadataAttributePrefix="Meta-" >
     ```

2. Add the attribute checker handler with the list of required attributes to Sessions (in the example below: `displayName`, `givenName`, `mail`, `cn`, `sn`, `eppn`, `schacHomeOrganization`, `schacHomeOrganizationType`). The attributes' names HAVE TO MATCH with those are defined on `attribute-map.xml`:
   * ``` text
     vim /etc/shibboleth/shibboleth2.xml
     ```

     ``` text
        ...
        <!-- Attribute Checker -->
        <Handler type="AttributeChecker" Location="/AttrChecker" template="attrChecker.html" attributes="displayName givenName mail cn sn eppn schacHomeOrganization schacHomeOrganizationType" flushSession="true"/>
     </Sessions>
     ```
     
     If you want to describe more complex scenarios with required attributes, operators such as "AND" and "OR" are available.
     ``` text
     <Handler type="AttributeChecker" Location="/AttrChecker" template="attrChecker.html" flushSession="true">
        <OR>
           <Rule require="displayName"/>
           <AND>
              <Rule require="givenName"/>
              <Rule require="surname"/>
           </AND>
        </OR>
      </Handler>
      ```

3. Add the following `<AttributeExtractor>` element under `<AttributeExtractor type="XML" validate="true" reloadChanges="false" path="attribute-map.xml"/>`:
   * ``` text
     vim /etc/shibboleth/shibboleth2.xml
     ```

     ``` text
     <!-- Extracts support information for IdP from its metadata. -->
     <AttributeExtractor type="Metadata" errorURL="errorURL" DisplayName="displayName"
                         InformationURL="informationURL" PrivacyStatementURL="privacyStatementURL"
                         OrganizationURL="organizationURL">
        <ContactPerson id="Technical-Contact"  contactType="technical" formatter="$EmailAddress" />
        <Logo id="Small-Logo" height="16" width="16" formatter="$_string"/>
     </AttributeExtractor>
     ```

4. Save and restart "shibd" service:
   * ``` text
     systemctl restart shibd.service
     ```
   
5. Customize Attribute Checker template:
   * ``` text
     cd /etc/shibboleth
     ```
   * ``` text
     cp attrChecker.html attrChecker.html.orig
     ```
   * ``` text
     wget https://raw.githubusercontent.com/CSCfi/shibboleth-attrchecker/master/attrChecker.html -O attrChecker.html
     ```
   * ``` text
     sed -i 's/SHIB_//g' /etc/shibboleth/attrChecker.html
     ```
   * ``` text
     sed -i 's/eduPersonPrincipalName/eppn/g' /etc/shibboleth/attrChecker.html
     ```
   * ``` text
     sed -i 's/Meta-Support-Contact/Meta-Technical-Contact/g' /etc/shibboleth/attrChecker.html
     ```
   * ``` text
     sed -i 's/supportContact/technicalContact/g' /etc/shibboleth/attrChecker.html
     ```
   * ``` text
     sed -i 's/support/technical/g' /etc/shibboleth/attrChecker.html
     ```

   There are three locations needing modifications to do on `attrChecker.html`:

   1. The pixel tracking link after the comment "PixelTracking". 
      The Image tag and all required attributes after the variable must be configured here. 
      After "`miss=`" define all required attributes you updated in `shibboleth2.xml` using shibboleth tagging. 
      
      Eg `<shibmlpifnot $attribute>-$attribute</shibmlpifnot>` (this echoes $attribute if it's not received by shibboleth).
      The attributes' names HAVE TO MATCH with those are defined on `attribute-map.xml`.
      
      This example uses "`-`" as a delimiter.
      
   2. The table showing missing attributes between the tags "`<!--TableStart-->`" and "`<!--TableEnd-->`". 
      You have to insert again all the same attributes as above.

      Define row for each required attribute (eg: `displayName` below)

      ```html
      <tr <shibmlpifnot displayName> class='warning text-danger'</shibmlpifnot>>
        <td>displayName</td>
        <td><shibmlp displayName /></td>
      </tr>
      ```

   3. The email template between the tags "<textarea>" and "</textarea>". After "The attributes that were not released to the service are:". 

      Again define all required attributes using shibboleth tagging like in section 1 ( eg: `<shibmlpifnot $attribute> * $attribute</shibmlpifnot>`).
      The attributes' names HAVE TO MATCH with those are defined on `attribute-map.xml`.
      Note that for SP identifier target URL is used instead of entityID. 
      There arent yet any tag for SP entityID so you can replace this target URL manually.

6. Enable Logging:
   * Create your `track.png` with into your DocumentRoot:
   
     ``` text
     echo "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg==" | base64 -d > /var/www/html/$(hostname -f)/track.png
     ```

   * Results into `/var/log/apache2/other_vhosts_access.log`:
   
   ```bash
   ./apache2/other_vhosts_access.log:193.206.129.66 - - [20/Sep/2018:15:05:07 +0000] "GET /track.png?idp=https://garr-idp-test.irccs.garr.it/idp/shibboleth&miss=-SHIB_givenName-SHIB_cn-SHIB_sn-SHIB_eppn-SHIB_schacHomeOrganization-SHIB_schacHomeOrganizationType HTTP/1.1" 404 637 "https://sp.example.org/Shibboleth.sso/AttrChecker?return=https%3A%2F%2Fsp.example.org%2FShibboleth.sso%2FSAML2%2FPOST%3Fhook%3D1%26target%3Dss%253Amem%253A43af2031f33c3f4b1d61019471537e5bc3fde8431992247b3b6fd93a14e9802d&target=https%3A%2F%2Fsp.example.org%2Fsecure%2F"
   ```

[TOC](#table-of-contents)

## Increase startup timeout

Shibboleth Documentation: https://wiki.shibboleth.net/confluence/display/SP3/LinuxSystemd

* ``` text
  sudo mkdir /etc/systemd/system/shibd.service.d
  ```
* ``` text
  echo -e '[Service]\nTimeoutStartSec=60min' | sudo tee /etc/systemd/system/shibd.service.d/timeout.conf
  ```
* ``` text
  sudo systemctl daemon-reload
  ```
* ``` text
  sudo systemctl restart shibd.service
  ```

[TOC](#table-of-contents)

## OPTIONAL - Maintain '`shibd`' working

1. Edit '`shibd`' init script:
   * ``` text
     vim /etc/init.d/shibd
     ```

     ```bash
     #...other lines...
     ### END INIT INFO

     # Import useful functions like 'status_of_proc' needed to 'status'
     . /lib/lsb/init-functions

     #...other lines...

     # Add 'status' operation
     status)
       status_of_proc -p "$PIDFILE" "$DAEMON" "$NAME" && exit 0 || exit $?
       ;;
     *)
       echo "Usage: $SCRIPTNAME {start|stop|restart|force-reload|status}" >&2
       exit 1
       ;;

     esac
     exit 0
     ```

2. Create a new watchdog for '`shibd`':
   * ``` text
     vim /etc/cron.hourly/watch-shibd.sh
     ```

     ```bash
     #! /bin/bash
     SERVICE=/etc/init.d/shibd
     STOPPED_MESSAGE="failed"

     if [[ "`$SERVICE status`" == *"$STOPPED_MESSAGE"* ]];
     then
       $SERVICE stop
       sleep 1
       $SERVICE start
     fi
     ```

3. Reload daemon:
   * ``` text
     systemctl daemon-reload
     ```

[TOC](#table-of-contents)

## Utility

* [The Mozilla Observatory](https://observatory.mozilla.org/):
  The Mozilla Observatory has helped over 240,000 websites by teaching developers, system administrators, and security professionals how to configure their sites safely and securely.

## Authors

### Original Author

 * Marco Malavolti (marco.malavolti@garr.it)

## Thanks

 * eduGAIN Wiki: For the original [How to configure Shibboleth SP attribute checker](https://wiki.geant.org/display/eduGAIN/How+to+configure+Shibboleth+SP+attribute+checker)

[TOC](#table-of-contents)
