# HOWTO Install and Configure Python Federation Feeder (pyFF)

The Python Federation Feeder (pyFF) is a simple, but reasonably complete, SAML metadata processor. 
It is intended to be used by anyone who needs to aggregate, validate, combine, transform, sign or publish SAML metadata.

## Table of Contents

1.  [Overview](#overview)
2.  [Requirements](#requirements)
    1.  [Hardware](#hardware)
    2.  [Software](#software)
3.  [Notes](#notes)
4.  [Configure the environment](#configure-the-environment)
5.  [Configure APT Mirror](#configure-apt-mirror)
6.  [Install Dependencies](#install-dependencies)
7.  [Install Apache Web Server](#install-apache-web-server)
8.  [Install PyFF](#install-pyff)
9.  [Configure PyFF](#configure-pyff)
    1. [Example 1 - Pipeline for a Test or a National Federation metadata](#example-1---pipeline-for-a-test-or-a-national-federation-metadata)
    2. [Example 2 - Pipeline for eduGAIN Downstream metadata](#example-2---pipeline-for-edugain-downstream-metadata)
    3. [Example 3 - Pipeline for eduGAIN Upstream metadata](#example-3---pipeline-for-edugain-upstream-metadata)
10. [Documentation](#documentation)
11. [Authors](#authors)
12. [Thanks](#thanks)

## Overview

This HOWTO is based on Python Federation Feeder (pyFF), 
which uses YAML config files to generate metadata. Let’s call those files `Pipelines`.

Depending on the nature of the Pipeline, additional files may be needed including things like…

* A list of metadata URLs.
* A set of files containing metadata URLs - eg XRD or MDSL files.
* XSL style sheet file.

## Requirements

### Hardware

-   CPU: 2 Core (64 bit)
-   RAM: 4 GB
-   HDD: 10 GB
-   OS: Debian 12 / Ubuntu 22.04

### Software

-   Apache Web Server (*\<= 2.4*)
-   Python (>= 3.6)

## Notes

This HOWTO will use `Vim` as text editor:

-   `Esc button + i` means "insert"
-   `Esc button + :w` means "write"
-   `Esc button + :q` means "quit"
-   `Esc button + :wq` means "write & quit"
-   `Esc button + /` means "search text"
-   `Esc button + u` means "undo"

[TOC](#table-of-contents)

## Configure the environment

1.  Become ROOT:

    ``` text
    sudo su -
    ```

2.  Be sure that your firewall **is not blocking** the traffic on port **80** for the Federation Metadata Server.

3.  Set the Federation Metadata Server hostname:

    **!!!ATTENTION!!!**: Replace `federation.example.org` with your Federation Metadata Server Full Qualified Domain Name and `<HOSTNAME>` with its hostname

    -   ``` text
        echo "<YOUR-SERVER-IP-ADDRESS> federation.example.org <HOSTNAME>" >> /etc/hosts
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

The Apache HTTP Server will be used to spread federetaion metadata to the institutions.

It is recommended to spread the Federation Metadata on HTTP. 

In this way the institutions will have to validate metadata with the federation metadata's signature key.

``` text
sudo apt install apache2
```

This HOWTO will save the metadata generated files into `/var/www/html/metadata` directory.

Remember to change the metadata path if it is used another directory to save the generated metadata.


[TOC](#table-of-contents)

## Install PyFF

1. Install requirements:

   * ``` text
     sudo su -
     ```

   * ``` text
     apt install build-essential python3-dev libxml2-dev libxslt1-dev libyaml-dev python3-pip python3-venv
     ```

2. Verify to use at least the minimum version of Python:

   * ``` text
     python3 --version
     ```

     (it has to return at least Python 3.6)

3. Create a system user `pyff` without SSH access:

   * ``` text
     sudo adduser --system --group --shell /bin/bash --home /opt/pyff pyff
     ```

4. Create PyFF Virtual Environment:

   * ``` text
     sudo su pyff
     ```

   * ``` text
     python3 -m venv /opt/pyff
     ```

5. Create needed directories for pipelines and signature credentials:

   * ``` text
     mkdir /opt/pyff/pipelines
     ```

   * ``` text
     mkdir /opt/pyff/sign-credentials
     ```

6. Install Pyff into the Virtual Environment created:

   * ``` text
     cd /opt/pyff
     ```

   * ``` text
     source bin/activate
     ```

   * ``` text
     pip install --upgrade pip
     ```

   * ``` text
     pip install --upgrade setuptools wheel
     ```
  
   * ``` text
     pip install --upgrade pyff
     ```

   * ``` text
     deactivate
     ```

     (exit from the Virtual Environment)

   * ``` text
     exit
     ```

     (logout for `pyff` user)

6. Add `pyFF` binary to the PATH of your server:

   * ``` text
     vim $HOME/.bash_profile
     ```

   * ``` text
     # set PATH for PyFF
     if [ -d "/opt/pyff" ]; then
        PATH="$PATH:/opt/pyff/bin"
     fi
     ```

   * ``` text
     vim $HOME/.bashrc
     ```

   * ``` text
     # Enable PyFF PATH on start
     if [ -f ~/.bash_profile ]; then
         . ~/.bash_profile
     fi
     ```

7. Test if it is working:

   * ``` text
     pyff --version
     ```

[TOC](#table-of-contents)

## Configure PyFF

If you want to generate signed metadata with PyFF you need a signing certificate.
This HOWTO will use a pair of self-signed certificate/private key (as an example) created with:

* ``` text
  sudo su pyff
  ```

* ``` text
  cd /opt/pyff
  ```

* ``` text
  openssl req -nodes -x509 -newkey rsa:4096 -keyout /opt/pyff/sign-credentials/sign.key -out /opt/pyff/sign-credentials/sign.crt -days 3650 -subj "/CN=$(hostname -f)"
  ```

This HOWTO assumes that you already have a process to collect metadata for your federations entities.

We suggest that you maintain a tree structure for each federation you manage (one can imagine you manage a **test** and a **national** federation):

``` text
/opt/pyff/pipelines/
├── test-fede/
│   ├── idps/
│   │   ├── preprod-test-idp.xml
│   │   └── test-idp.xml
│   ├── sps/
│   │   ├── preprod-test-sp.xml
│   │   └── test-sp.xml
│   └── pipeline-test-fede.yml
├── national-fede/
│   ├── idps/
│   │   ├── idp-university-X.xml
│   │   └── idp-university-X.xml
│   ├── sps/
│   │   ├── filesender.xml
│   │   └── university-X-moodle.xml
│   └── pipeline-national-fede.yml
└── edugain/
    ├── idps/
    │   ├── idp-university-X.xml
    │   └── idp-university-Y.xml
    ├── sps/
    │   └── university-X-moodle.xml
    ├── pipeline-edugain-downstream.yml
    └── pipeline-edugain-upstream.yml
```

In the following example you have 3 federations:
1. A test federation
2. A national federation
3. eduGAIN, which is a bit special, because you have to produce 2 metadata files:

   - You have to produce a metadata stream of entities from YOUR national federation that will be consumed by eduGAIN to merge them in the global eduGAIN metadata stream (MDS)
   - You have to produce a metadata stream of entities from eduGAIN, based on eduGAIN MDS – excluding entities from YOUR national federation

The sample metadata files are provided into `config-files/pyff/sample` directory.

[TOC](#table-of-contents)

### Example 1 - Pipeline for a Test or a National Federation metadata

The pipeline for test and national federation looks quite the same. 
Here’s the sample for test federation.

1. Create the Pipeline for Test Federation metadata:

   * ``` text
     vim /opt/pyff/pipelines/test-fede/pipeline-test-fede.yml
     ```

   * ``` yaml
     ---
     # Pipeline to sign and publish
     - load fail_on_error True filter_invalid True:
       # IDPS
       - /opt/pyff/pipelines/test-fede/idps/test-idp.xml
       - /opt/pyff/pipelines/test-fede/idps/preprod-test-idp.xml
       # - /base/path/for/your/metadata/files/test-fede/test-idp-2.xml
       # SPS
       - /opt/pyff/pipelines/test-fede/sps/test-sp.xml
       - /opt/pyff/pipelines/test-fede/sps/preprod-test-sp.xml
       # - /base/path/for/your/metadata/files/test-fede/test-sp-2.xml
     - select:
     - reginfo:
         authority: https://nren.foo/
         policy:
           en: https://nren.foo/policy.html
     - finalize:
         cacheDuration: PT1H
         validUntil: P9D
         Name: https://nren.foo/test
     - sign:
         key: /opt/pyff/sign-credentials/sign.key
         cert: /opt/pyff/sign-credentials/sign.crt
     - xslt:
         stylesheet: pp.xsl
     # Publish directly to web server
     - publish: /var/www/html/metadata/test-fede-metadata.xml
     - stats
     ```

2. Run the Pipeline: 

   * ``` text
     pyff /opt/pyff/pipelines/test-fede/pipeline-test-fede.yml
     ```

Explanation:

| Section  | Description                                                                |
|----------|----------------------------------------------------------------------------|
| load     | Enumerates all metadata files to include in the federation                 |
| select   | Filtering                                                                  |
| reginfo  | Sets registration info extension on EntityDescription element              |
| finalize | Sets validity and name of the metadata stream                              |
| sign     | Signs the file                                                             |
| xslt     | Applies a XSL transformation.pp.xsl is builtin Pyff and makes pretty print |
| publish  | Tells where to put the result                                              |
| stats    | Prints stats on the generated file                                         |

[TOC](#table-of-contents)

### Example 2 - Pipeline for eduGAIN Downstream metadata

eduGAIN downstream metadata is targeted at the entities of your national federation.
Thus, you have to exclude entities from your national federation that are already present in eduGAIN as they are already known by your entities.

1. Create the Pipeline for eduGAIN Downstream metadata:

   * ``` text
     vim /opt/pyff/pipelines/edugain/pipeline-edugain-downstream.yml
     ```

   * ``` yaml
     ---
     - load: http://mds.edugain.org/feed-sha256.xml
     - select
     - filter:
       # Exclude entities from your national fede because entities from your federation already know about them
       - "!//md:EntityDescriptor[not(md:Extensions/mdrpi:RegistrationInfo/@registrationAuthority='https://nren.foo/')]"
     - finalize:
         cacheDuration: PT1H
         validUntil: P9D
         Name: https://nren.foo/edugain
     - sign:
         key: /opt/pyff/sign-credentials/sign.key
         cert: /opt/pyff/sign-credentials/sign.crt
     - xslt:
         stylesheet: pp.xsl
     - publish: /var/www/html/metadata/edugain-metadata-downstream.xml
     - stats
     ```

2. Run the Pipeline: 

   * ``` text
     pyff /opt/pyff/pipelines/edugain/pipeline-edugain-downstream.yml
     ```

Additional Explanation:

| Section | Description                                                                                                                                                                          |
|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| filter  | Allows to filter entities.<br>The filter in the sample, filters using the **RegistrationAuthority**, <br>which you configure in the pipeline in the **reginfo.authority** parameter. |
| reginfo | Not used for eduGAIN downstream, as you have to keep this information unchanged,<br>so other entities can know which NREN declares this entity.                                      |

[TOC](#table-of-contents)

### Example 3 - Pipeline for eduGAIN Upstream metadata

The pipeline for upstream eduGAIN metadata has to generate metadata for the entities that you, as a Federation Operator:
   - want to make accessible to other countries (for SPs)
   - want to be used to access international resources (for IDPs)

It looks much like test/national federation pipeline.

1. Create the Pipeline for eduGAIN Upstream metadata:

   * ``` text
     vim /opt/pyff/pipelines/edugain/pipeline-edugain-upstream.yml
     ```

   * ``` yaml
     ---
     # Pipeline to sign and publish for edugain
     - load fail_on_error True filter_invalid True:
       # IDPS
       - /base/path/for/your/metadata/files/edugain/idps/idp-university-X.xml
       - /base/path/for/your/metadata/files/edugain/idps/idp-university-Y.xml
       # SPS
       - /base/path/for/your/metadata/files/edugain/sps/university-X-moodle.xml
     - select:
     - reginfo:
         authority: https://nren.foo/
         policy:
           en: https://nren.foo/policy.html
     - finalize:
         cacheDuration: PT1H
         validUntil: P9D
         Name: https://nren.foo/test
     - sign:
         key: /opt/pyff/sign-credentials/sign.key
         cert: /opt/pyff/sign-credentials/sign.crt
     - xslt:
         stylesheet: pp.xsl
     # Publish directly to web server
     - publish: /var/www/html/metadata/edugain-metadata-upstream.xml
     - stats
     ```

2. Run the Pipeline: 

   * ``` text
     pyff /opt/pyff/pipelines/edugain/pipeline-edugain-upstream.yml
     ```

[TOC](#table-of-contents)

## Cron Job

The Cron Job wil be used to update the metadata stream generated by Pyff

This HOWTO will use an hourly cron job for `test-fede` metadata stream.

   * ``` text
     sudo su -
     ```

   * ``` text
     bash -c 'cat > /etc/cron.hourly/update-test-fede-metadata <<EOF
     #!/bin/bash
     
     pyff /opt/pyff/pipelines/test-fede/pipeline-test-fede.yml
     EOF'     
     ```

   * ``` text
     chmod +x /etc/cron.hourly/update-test-fede-metadata
     ```

     (remember to remove the line `- stats` that will produce the statistics data on stdout)

## Documentation

- [PyFF Homepage](https://pyff.readthedocs.io/)
- [PyFF Github Repository](https://github.com/IdentityPython/pyFF)

[TOC](#table-of-contents)

## Authors

### Original Author

 * Marco Malavolti (marco.malavolti@garr.it)

## Thanks

 * Anass Chabli

[TOC](#table-of-contents)
