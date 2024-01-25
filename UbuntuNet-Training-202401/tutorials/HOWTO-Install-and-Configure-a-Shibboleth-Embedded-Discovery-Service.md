# HOWTO Install and Configure a Shibboleth Embedded Discovery Service

The Embedded Discovery Service (EDS) allows a Service Provider to run a discovery service within their own site. 
As such the discovery service can look like any other page on the site and thus not be as jarring to a user as being redirected to a totally different, third-party, discovery service site.
The EDS is a set of Javascript and CSS files, so installing it and using it is straight forward and does not require any additional software. 
Note: you must already have an installed and configured Shibboleth Service Provider, V2.4+, in order to use the EDS.

## Table of Contents

1. [Requirements](#requirements)
    1.  [Hardware](#hardware)
    2.  [Software](#software)
2. [Notes](#notes) 
3. [Install Shibboleth Embedded Discovery Service on Service Provider](#install-shibboleth-embedded-discovery-service-on-service-provider)
4. [Enable Shibboleth EDS](#enable-shibboleth-eds)
5. [Configuration](#configuration)
6. [Whitelist - How to allow IdPs to access the federated resource](#whitelist---how-to-allow-idps-to-access-the-federated-resource)
  1. [How to allow the access to IdPs by specifying their entityID](#how-to-allow-the-access-to-idps-by-specifying-their-entityid)
  2. [How to allow the access to IdPs that support a specific Entity Category](#how-to-allow-the-access-to-idps-that-support-a-specific-entity-category)
  3. [How to allow the access to IdPs that support SIRTFI](#how-to-allow-the-access-to-idps-that-support-sirtfi)
7. [Blacklist - How to disallow IdPs to access the federated resource](#blacklist---how-to-disallow-idps-to-access-the-federated-resource)
  1. [How to disallow the access to IdPs by specifying their entityID](#how-to-disallow-the-access-to-idps-by-specifying-their-entityid)
  2. [How to disallow the access to IdPs that support a specific Entity Category](#how-to-disallow-the-access-to-idps-that-support-a-specific-entity-category)
8. [Best Practices to follow to maximize the access to the resource](#best-practices-to-follow-to-maximize-the-access-to-the-resource)
9. [Authors](#authors)
10. [Credits](#credits)

## Requirements

### Hardware

* OS: Debian 12 / Ubuntu 22.04

### Software

* Apache Web Server (>= 2.4)
* A working Shibboleth Service Provider (>= 2.4)

## Notes

This HOWTO uses `example.org` and `shib-sp.example.org` as example values.

Please remember to **replace all occurencences** of:

-   the `example.org` value with the IdP domain name
-   the `idp.example.org` value with the Full Qualified Domain Name of the Identity Provider.
-   the `shib-sp.example.org` value with the Full Qualified Domain Name of the Service Provider.

This HOWTO will use `Vim` as text editor:
-   `Esc button + i` means "insert"
-   `Esc button + :w` means "write"
-   `Esc button + :q` means "quit"
-   `Esc button + :wq` means "write & quit"
-   `Esc button + /` means "search text"

[TOC](#table-of-contents)

## Install Shibboleth Embedded Discovery Service on Service Provider

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2. Download and install the latest version of Shibboleth Embedded Discovery Service:

   -  ``` text
      sudo su -
      ```
   -  ``` text
      cd /usr/local/src
      ```
   -  ``` text
      wget https://shibboleth.net/downloads/embedded-discovery-service/latest/shibboleth-embedded-ds-1.2.2.tar.gz -O shibboleth-eds.tar.gz
      ```
   -  ``` text
      tar xzf shibboleth-eds.tar.gz
      ```
   -  ``` text
      cd shibboleth-embedded-ds-1.2.2
      ```
   -  ``` text
      apt install make
      ```
   -  ``` text
      make install
      ```

3. Enable Discovery Service Web Page:
   
   - ``` text
     mv /etc/shibboleth-ds/shibboleth-ds.conf /etc/apache2/conf-available/shibboleth-ds.conf
     ```

   - ``` text
     a2enconf shibboleth-ds.conf
     ```

4. Restart Apache to load the new web site:

   ``` text
   systemctl restart apache2.service
   ```

[TOC](#table-of-contents)

## Enable Shibboleth EDS

1. Update `shibboleth2.xml` file to point to the new Discovery Service page:

   * ``` text
     vim /etc/shibboleth/shibboleth2.xml
     ```
 
     ```xml
     <SSO discoveryProtocol="SAMLDS" 
          discoveryURL="https://shib-sp.example.org/shibboleth-ds/index.html">
        SAML2
     </SSO>

     <!-- SAML and local-only logout. -->
     <Logout>SAML2 Local</Logout>

     <!-- ...other things ... -->

     <!-- JSON feed of discovery information. -->
     <Handler type="DiscoveryFeed" Location="/DiscoFeed"/>
     ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

## Configuration

The behavior of the Shibboleth Embedded Discovery Service is governed by the `IdPSelectUIParms` class, which is located in `/etc/shibboleth-ds/idpselect_config.js`.
In the most of cases you have to modify only this file to change the behaviour of Discovery Service:

``` text
vim /etc/shibboleth-ds/idpselect_config.js
```

Make sure to amend `this.returnWhiteList` to reflect your server name.

Find here the EDS Configuration Options: https://wiki.shibboleth.net/confluence/display/EDS10/3.+Configuration

[TOC](#table-of-contents)

## Whitelist - How to allow IdPs to access the federated resource

### How to allow the access to IdPs by specifying their entityID

1. Modify Shibboleth Service Provider configuration file `shibboleth2.xml`:

   ``` text
   vim /etc/shibboleth/shibboleth2.xml
   ```

   ```xml
   <MetadataProvider type="XML"
                     uri="###-URL-TO-FEDERATION-METADATA-XML-STREAM-###"
                     backingFilePath="federation-metadata.xml">
      <MetadataFilter type="Signature" certificate="/etc/shibboleth/###-METADATA-SIGNATURE-KEY-PROVIDED-BY-FEDERATION-###"/>
      <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
      <MetadataFilter type="Whitelist">
          <Include>https://entityid.idp1.allowed.it/shibboleth</Include>
          <Include>https://entityid.idp2.allowed.it/shibboleth</Include>
          <Include>https://entityid.idp3.allowed.it/shibboleth</Include>
      </MetadataFilter>
   </MetadataProvider>
   ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

### How to allow the access to IdPs that support a specific Entity Category

1. Modify Shibboleth Service Provider configuration file `shibboleth2.xml`:

   ``` text
   vim /etc/shibboleth/shibboleth2.xml
   ```
   
   ```xml
   <MetadataProvider type="XML"
                     uri="###-URL-TO-FEDERATION-METADATA-XML-STREAM-###"
                     backingFilePath="federation-metadata.xml">
      <MetadataFilter type="Signature" certificate="/etc/shibboleth/###-METADATA-SIGNATURE-KEY-PROVIDED-BY-FEDERATION-###"/>
      <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
      <MetadataFilter type="Whitelist" matcher="EntityAttributes">
          <saml:Attribute Name="http://macedir.org/entity-category"
                          NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
              <saml:AttributeValue>http://refeds.org/category/research-and-scholarship</saml:AttributeValue>
          </saml:Attribute>
      </MetadataFilter>
   </MetadataProvider>
   ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

### How to allow the access to IdPs that support SIRTFI

1. Modify Shibboleth Service Provider configuration file `shibboleth2.xml`:

   ``` text
   vim /etc/shibboleth/shibboleth2.xml
   ```

   ```xml
   <MetadataProvider type="XML"
                     uri="###-URL-TO-FEDERATION-METADATA-XML-STREAM-###"
                     backingFilePath="federation-metadata.xml">
       <MetadataFilter type="Signature" certificate="/etc/shibboleth/###-METADATA-SIGNATURE-KEY-PROVIDED-BY-FEDERATION-###"/>
       <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
       <MetadataFilter type="Whitelist" matcher="EntityAttributes">
           <saml:Attribute Name="urn:oasis:names:tc:SAML:attribute:assurancecertification"
                           NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
               <saml:AttributeValue>https://refeds.org/sirtfi</saml:AttributeValue>
           </saml:Attribute>
       </MetadataFilter>
   </MetadataProvider>
   ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

## Blacklist - How to disallow IdPs to access the federated resource

### How to disallow the access to IdPs by specifying their entityID

1. Modify Shibboleth Service Provider configuration file `shibboleth2.xml`:

   ``` text
   vim /etc/shibboleth/shibboleth2.xml
   ```
  
   ```xml
   <MetadataProvider type="XML"
                     uri="###-URL-TO-FEDERATION-METADATA-XML-STREAM-###"
                     backingFilePath="federation-metadata.xml">
      <MetadataFilter type="Signature" certificate="/etc/shibboleth/###-METADATA-SIGNATURE-KEY-PROVIDED-BY-FEDERATION-###"/>
      <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
      <MetadataFilter type="Blacklist">
          <Include>https://entityid.idp1.denied.it/shibboleth</Include>
          <Include>https://entityid.idp2.denied.it/shibboleth</Include>
          <Include>https://entityid.idp3.denied.it/shibboleth</Include>
      </MetadataFilter>
   </MetadataProvider>
   ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

### How to disallow the access to IdPs that support a specific Entity Category

1. Modify Shibboleth Service Provider configuration file `shibboleth2.xml`:

   ``` text
   vim /etc/shibboleth/shibboleth2.xml
   ```

   ```xml
   <MetadataProvider type="XML"
                     uri="###-URL-TO-FEDERATION-METADATA-XML-STREAM-###"
                     backingFilePath="federation-metadata.xml">
      <MetadataFilter type="Signature" certificate="/etc/shibboleth/###-METADATA-SIGNATURE-KEY-PROVIDED-BY-FEDERATION-###"/>
      <MetadataFilter type="RequireValidUntil" maxValidityInterval="864000" />
      <MetadataFilter type="Blacklist" matcher="EntityAttributes">
          <saml:Attribute Name="http://macedir.org/entity-category"
                          NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri">
              <saml:AttributeValue>https://federation.renater.fr/scope/commercial</saml:AttributeValue>
          </saml:Attribute>
      </MetadataFilter>
   </MetadataProvider>
   ```

2. Restart `shibd` service:

   ``` text
   systemctl restart shibd.service
   ```

[TOC](#table-of-contents)

## Best Practices to follow to maximize the access to the resource

* [REFEDS Discovery Guide](https://discovery.refeds.org/)

## Authors

### Original Author

* Marco Malavolti (marco.malavolti@garr.it)
 
## Credits

* [Consortium Shibboleth](https://shibboleth.net/)
* [REFEDS Discovery Guide](https://discovery.refeds.org/)

[TOC](#table-of-contents)
