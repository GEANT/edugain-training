# HOWTO Install and Configure OpenLDAP for federated access

## Table of Contents

1.  [Requirements](#requirements)
2.  [Notes](#notes)
3.  [Utility](#utility)
4.  [Installation](#installation)
5.  [Configuration](#configuration)
6.  [Password Policies](#password-policies)
7.  [Authors](#authors)

## Requirements

-   Tested OS:
    -   Debian 11 (Buster)
    -   Ubuntu 20.04 (Local Fossa)
    -   Debian 12 (Bookworm)
    -   Ubuntu 22.04 (Jammy Jellyfish)

## Notes

This HOWTO uses `example.org` and `ldap.example.org` as example values.

Please remember to **replace all occurencences** of:

-   the `example.org` value with the domain name of your institution
-   the `ldap.example.org` value with the Full Qualified Domain Name of the OpenLDAP server.

This HOWTO will use `Vim` as text editor:
-   `Esc button + i` means "insert"
-   `Esc button + :w` means "write"
-   `Esc button + :q` means "quit"
-   `Esc button + :wq` means "write & quit"
-   `Esc button + /` means "search text"

## Utility

-   Simple Bash script useful to convert a Domain Name into a
    Distinguished Name of LDAP:
    [domain2dn.sh](https://github.com/GEANT/edugain-training/blob/main/UbuntuNet-Training-202401/scripts/domain2dn.sh)

## Installation

1.  System Update:

    -   ``` text
        sudo apt update ; sudo apt upgrade
        ```

2.  Install needed packages to automate the SLAPD installation:

    -   ``` text
        sudo apt install debconf-utils
        ```

3.  Automate SLAPD installation (Change all `_CHANGEME` values):

    -   ``` text
        sudo vim /root/debconf-slapd.conf
        ```

        ``` bash
        slapd slapd/password1 password <LDAP-ROOT-PW_CHANGEME>
        slapd slapd/password2 password <LDAP-ROOT-PW_CHANGEME>
        slapd slapd/move_old_database boolean true
        slapd slapd/domain string <INSTITUTE-DOMAIN_CHANGEME>
        slapd shared/organization string <ORGANIZATION-NAME_CHANGEME>
        slapd slapd/no_configuration boolean false
        slapd slapd/purge_database boolean false
        slapd slapd/allow_ldap_v2 boolean false
        slapd slapd/backend select MDB
        ```

    -   ``` text
        sudo cat /root/debconf-slapd.conf | sudo debconf-set-selections
        ```

    **NOTES**: This HOWTO considers the following example values that
    have to be changed as your needs:

    -   `<LDAP-ROOT-PW_CHANGEME>` ==\> `ciaoldap`
    -   `<INSTITUTE-DOMAIN_CHANGEME>` ==\> `example.org`
    -   `<ORGANIZATION-NAME_CHANGEME>` ==\> `Example Org`

4.  Install required package:

    -   ``` text
        sudo apt install slapd ldap-utils ldapscripts rsyslog
        ```

5.  Set the OpenLDAP hostname:

    **!!!ATTENTION!!!**: Replace `ldap.example.org` with your OpenLDAP's
    server Full Qualified Domain Name and `<HOSTNAME>` with its
    hostname.

    -   ``` text
        echo "<YOUR-SERVER-IP-ADDRESS> ldap.example.org <HOSTNAME>" >> /etc/hosts
        ```

    -   ``` text
        hostnamectl set-hostname <HOSTNAME>
        ```

6.  Create Self Signed Certificate/Private Key (4096 bit - 3 years before expiration):

    -   ``` text
        sudo openssl req -newkey rsa:4096 -x509 -nodes -out /etc/ldap/$(hostname -f).crt -keyout /etc/ldap/$(hostname -f).key -days 1095 -subj "/CN=$(hostname -f)"
        ```

    -   ``` text
        sudo chown openldap:openldap /etc/ldap/$(hostname -f).crt
        ```

    -   ``` text
        sudo chown openldap:openldap /etc/ldap/$(hostname -f).key
        ```

7.  Enable SSL for LDAP:

    -   ``` text
        sudo sed -i "s/TLS_CACERT.*/TLS_CACERT\t\/etc\/ldap\/$(hostname -f).crt/g" /etc/ldap/ldap.conf
        ```

    -   ``` text
        sudo chown openldap:openldap /etc/ldap/ldap.conf
        ```

    -   ``` text
        sudo echo -e "TLS_CACERT\t/etc/ldap/$(hostname -f).crt" >> /etc/ldap/ldap.conf
        ```

    -   ``` text
        sudo chown openldap:openldap /etc/ldap/ldap.conf
        ```

8.  Restart OpenLDAP:

    -   ``` text
        sudo service slapd restart
        ```

## Configuration

1.  Create the `scratch` directory:

    -   ``` text
        sudo mkdir /etc/ldap/scratch
        ```

2.  Configure LDAP for SSL:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/olcTLS.ldif <<EOF
        dn: cn=config
        changetype: modify
        replace: olcTLSCACertificateFile
        olcTLSCACertificateFile: /etc/ldap/$(hostname -f).crt
        -
        replace: olcTLSCertificateFile
        olcTLSCertificateFile: /etc/ldap/$(hostname -f).crt
        -
        replace: olcTLSCertificateKeyFile
        olcTLSCertificateKeyFile: /etc/ldap/$(hostname -f).key
        EOF`
        ```

    -   ``` text
        sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/olcTLS.ldif
        ```

3.  Create the 3 main Organizational Unit (OU), `people`, `groups` and
    `system`.

    *Example:* if the domain name is `example.org` than the distinguish
    name will be `dc=example,dc=org`:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name and
    `<LDAP-ROOT-PW_CHANGEME>` with the LDAP ROOT password!

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/add_ou.ldif <<EOF
        dn: ou=people,dc=example,dc=org
        objectClass: organizationalUnit
        objectClass: top
        ou: people

        dn: ou=groups,dc=example,dc=org
        objectClass: organizationalUnit
        objectClass: top
        ou: groups

        dn: ou=system,dc=example,dc=org
        objectClass: organizationalUnit
        objectClass: top
        ou: system
        EOF'
        ```

    -   ``` text
        sudo ldapadd -x -D 'cn=admin,dc=example,dc=org' -w '<LDAP-ROOT-PW_CHANGEME>' -H ldapi:/// -f /etc/ldap/scratch/add_ou.ldif
        ```

    -   ``` text
        sudo ldapsearch -x -b 'dc=example,dc=org'
        ```

4.  Create the `idpuser` needed to perform "*Bind and Search*"
    operations:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name,
    `<LDAP-ROOT-PW_CHANGEME>` with the LDAP ROOT password and
    `<INSERT-HERE-IDPUSER-PW>` with password for the `idpuser` user!

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/add_idpuser.ldif <<EOF
        dn: cn=idpuser,ou=system,dc=example,dc=org
        objectClass: inetOrgPerson
        cn: idpuser
        sn: idpuser
        givenName: idpuser
        userPassword: <INSERT-HERE-IDPUSER-PW>
        EOF'
        ```

    -   ``` text
        sudo ldapadd -x -D 'cn=admin,dc=example,dc=org' -w '<LDAP-ROOT-PW_CHANGEME>' -H ldapi:/// -f /etc/ldap/scratch/add_idpuser.ldif
        ```

5.  Configure OpenLDAP ACL to allow `idpuser` to perform **search**
    operation on the directory:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   Check which configuration your directory has:

        ``` text
        sudo ldapsearch  -Y EXTERNAL -H ldapi:/// -b cn=config 'olcDatabase={1}mdb'
        ```

    -   Configure ACL for `idpuser` with:

        -   ``` text
            sudo bash -c 'cat > /etc/ldap/scratch/olcAcl.ldif <<EOF
            dn: olcDatabase={1}mdb,cn=config
            changeType: modify
            replace: olcAccess
            olcAccess: {0}to * by dn.exact=gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth manage by * break
            olcAccess: {1}to attrs=userPassword by self write by anonymous auth by dn="cn=admin,dc=example,dc=org" write by * none
            olcAccess: {2}to dn.base="" by anonymous auth by * read
            olcAccess: {3}to dn.base="cn=Subschema" by * read
            olcAccess: {4}to * by dn.exact="cn=idpuser,ou=system,dc=example,dc=org" read by anonymous auth by self read
            EOF'
            ```

        -   ``` text
            sudo ldapadd  -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/olcAcl.ldif
            ```

6.  Check that `idpuser` can search other users (when users exist):

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   ``` text
        sudo ldapsearch -x -D 'cn=idpuser,ou=system,dc=example,dc=org' -w '<INSERT-HERE-IDPUSER-PW>' -b 'ou=people,dc=example,dc=org'
        ```

7.  Install needed schemas (eduPerson, SCHAC, Password Policy):

    -   ``` text
        sudo wget https://raw.githubusercontent.com/REFEDS/eduperson/master/schema/openldap/eduperson.ldif -O /etc/ldap/schema/eduperson.ldif
        ```

    -   ``` text
        sudo wget https://raw.githubusercontent.com/REFEDS/SCHAC/main/schema/openldap.ldif -O /etc/ldap/schema/schac.ldif
        ```

    -   ``` text
        sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/eduperson.ldif
        ```

    -   ``` text
        sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/schac.ldif
        ```

    -   ``` text
        sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/ppolicy.ldif
        ```

        for Ubuntu 22.04 LTS and Debian 12 it does not exist! Follow
        [Password Policies](#password-policies).

        and verify presence of the new `schac`, `eduPerson` and
        `ppolicy` schemas with:

    -   ``` text
        sudo ldapsearch -Q -LLL -Y EXTERNAL -H ldapi:/// -b 'cn=schema,cn=config' dn
        ```

        for Ubuntu \>= 22.04 or Debian 12 follow [Password
        Policies](#password-policies).

8.  Add MemberOf Configuration to OpenLDAP directory:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/add_memberof.ldif <<EOF
        dn: cn=module,cn=config
        cn: module
        objectClass: olcModuleList
        olcModuleLoad: memberof
        olcModulePath: /usr/lib/ldap

        dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
        objectClass: olcConfig
        objectClass: olcMemberOf
        objectClass: olcOverlayConfig
        objectClass: top
        olcOverlay: memberof
        olcMemberOfDangling: ignore
        olcMemberOfRefInt: TRUE
        olcMemberOfGroupOC: groupOfNames
        olcMemberOfMemberAD: member
        olcMemberOfMemberOfAD: memberOf
        EOF'
        ```

    -   ``` text
        sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/add_memberof.ldif
        ```

9.  Improve performance:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/olcDbIndex.ldif <<EOF
        dn: olcDatabase={1}mdb,cn=config
        changetype: modify
        replace: olcDbIndex
        olcDbIndex: objectClass eq
        olcDbIndex: member eq
        olcDbIndex: cn pres,eq,sub
        olcDbIndex: ou pres,eq,sub
        olcDbIndex: uid pres,eq
        olcDbIndex: entryUUID eq
        olcDbIndex: sn pres,eq,sub
        olcDbIndex: mail pres,eq,sub
        EOF'
        ```

    -   ``` text
        sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/olcDbIndex.ldif
        ```

10. Configure Logging:

    -   ``` text
        sudo mkdir /var/log/slapd
        ```

    -   ``` text
        sudo bash -c 'cat > /etc/rsyslog.d/99-slapd.conf <<EOF
        local4.* /var/log/slapd/slapd.log
        EOF'
        ```

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/olcLogLevelStats.ldif <<EOF
        dn: cn=config
        changeType: modify
        replace: olcLogLevel
        olcLogLevel: stats
        EOF'
        ```

    -   ``` text
        sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/olcLogLevelStats.ldif
        ```

    -   ``` text
        sudo service rsyslog restart
        ```

    -   ``` text
        sudo service slapd restart
        ```

11. Configure openLDAP olcSizeLimit:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/olcSizeLimit.ldif <<EOF
        dn: cn=config
        changetype: modify
        replace: olcSizeLimit
        olcSizeLimit: unlimited

        dn: olcDatabase={-1}frontend,cn=config
        changetype: modify
        replace: olcSizeLimit
        olcSizeLimit: unlimited
        EOF'
        ```

    -   ``` text
        sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/olcSizeLimit.ldif
        ```

12. Add your first user:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/user1.ldif <<EOF
        # USERNAME: user1 , PASSWORD: ciaouser1
        # Generate a new password with: sudo slappasswd -s <newPassword>
        dn: uid=user1,ou=people,dc=example,dc=org
        changetype: add
        objectClass: inetOrgPerson
        objectClass: eduPerson
        uid: user1
        sn: User1
        givenName: Test
        cn: Test User1
        displayName: Test User1
        preferredLanguage: it
        userPassword: {SSHA}u5tYgO6iVerMuuMJBsYnPHM+70ammhnj
        mail: test.user1@example.org
        eduPersonAffiliation: student
        eduPersonAffiliation: staff
        eduPersonAffiliation: member
        eduPersonEntitlement: urn:mace:dir:entitlement:common-lib-terms
        eduPersonEntitlement: urn:mace:terena.org:tcs:personal-user
        EOF'
        ```

    -   ``` text
        sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/user1.ldif
        ```

13. Check that `idpuser` can find the inserted `user1`:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   ``` text
        sudo ldapsearch -x -D 'cn=idpuser,ou=system,dc=example,dc=org' -w '<INSERT-HERE-IDPUSER-PW>' -b 'uid=user1,ou=people,dc=example,dc=org'
        ```

14. Check that LDAP has TLS (`anonymous` MUST BE returned):

    -   ``` text
        sudo ldapwhoami -H ldap:// -x -ZZ
        ```

15. Make `mail`, `eduPersonPrincipalName` and `schacPersonalUniqueID` as
    unique:

    -   Load `unique` module:
        -   ``` text
            sudo bash -c 'cat > /etc/ldap/scratch/loadUniqueModule.ldif <<EOF
            dn: cn=module{0},cn=config
            changetype: modify
            add: olcModuleLoad
            olcModuleload: unique
            EOF'
            ```

        -   ``` text
            sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/loadUniqueModule.ldif
            ```
    -   Configure `mail`, `eduPersonPrincipalName` and
        `schacPersonalUniqueID` as unique:
        -   ``` text
            sudo bash -c 'cat > /etc/ldap/scratch/mail_ePPN_sPUI_unique.ldif <<EOF
            dn: olcOverlay=unique,olcDatabase={1}mdb,cn=config
            objectClass: olcOverlayConfig
            objectClass: olcUniqueConfig
            olcOverlay: unique
            olcUniqueAttribute: mail
            olcUniqueAttribute: schacPersonalUniqueID
            olcUniqueAttribute: eduPersonPrincipalName
            EOF'
            ```

        -   ``` text
            sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/mail_ePPN_sPUI_unique.ldif
            ```

16. Disable Anonymous bind:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/disableAnonymoysBind.ldif <<EOF
        dn: cn=config
        changetype: modify
        add: olcDisallows
        olcDisallows: bind_anon

        dn: olcDatabase={-1}frontend,cn=config
        changetype: modify
        add: olcRequires
        olcRequires: authc
        EOF'
        ```

    -   ``` text
        sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/ldap/scratch/disableAnonymoysBind.ldif
        ```

## Password Policies

1.  Load Password Policy module:

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/load-ppolicy-mod.ldif <<EOF
        dn: cn=module{0},cn=config
        changetype: modify
        add: olcModuleLoad
        olcModuleLoad: ppolicy.la
        EOF'
        ```

    -   ``` text
        sudo ldapadd -Y EXTERNAL -H ldapi:/// -f load-ppolicy-mod.ldif
        ```

2.  Create Password Policies OU Container:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/policies-ou.ldif <<EOF
        dn: ou=policies,dc=example,dc=org
        objectClass: organizationalUnit
        objectClass: top
        ou: policies
        EOF'
        ```

3.  Create OpenLDAP Password Policy Overlay DN:

    **Be carefull!** Replace `dc=example,dc=org` with distinguish name
    ([DN](https://ldap.com/ldap-dns-and-rdns/)) of your domain name!

    -   ``` text
        sudo bash -c 'cat > /etc/ldap/scratch/ppolicy-overlay.ldif <<EOF
        dn: olcOverlay=ppolicy,olcDatabase={1}mdb,cn=config
        objectClass: olcOverlayConfig
        objectClass: olcPPolicyConfig
        olcOverlay: ppolicy
        olcPPolicyDefault: cn=default,ou=policies,dc=example,dc=org
        olcPPolicyHashCleartext: TRUE
        EOF'
        ```

### Authors

#### Original Author

Marco Malavolti (<marco.malavolti@garr.it>)
