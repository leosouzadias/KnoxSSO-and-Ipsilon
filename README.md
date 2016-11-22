# Configuring Apache Knox SSO with Ipsilon using SAML2
This tutorial will provide the steps to install, configure and integrate Ipsilon with Knox SSO, using SAML2. 

Knox Server will be a Service Provides of Ipsilon.

## Requirements
1. Apache Knox installed
2. Apache Ambari Server installed (for Ambari UI SSO)
3. Apache Ranger installed (from Ranger UI SSO)
4. IPA installed and configured


## Installation

1. Install and Configure Ipsilon Server with SAML2 support:
    ```
    ipsilon-server-install --saml2=yes --form=yes --gssapi=yes --ipa=yes  --info-sssd=yes
    ```
2. Patch the Ipsilon Server to fix NameIDPolicy bug (https://pagure.io/ipsilon/pull-request/44). Patch is not included on version 1.0.0 of Ipsilon that can be downloaded from EPEL.

    ```diff
    From e23eead22c21258c3a0ef22a65f8e1aebc115b77 Mon Sep 17 00:00:00 2001
    From: Rob Crittenden <rcritten@redhat.com>
    Date: Oct 21 2015 14:52:38 +0000
    Subject: Don't crash if no NameIdPolicy is requested


    This fixes two problems:

    1. Logging was done before a None check was completed
    2. The None check was insufficient because the whole object
       could be None

    Signed-off-by: Rob Crittenden <rcritten@redhat.com>

    https://fedorahosted.org/ipsilon/ticket/189

    ---

    diff --git a/ipsilon/providers/saml2/provider.py b/ipsilon/providers/saml2/provider.py
    index 6cbf5ab..6d46ad2 100644
    --- a/ipsilon/providers/saml2/provider.py
    +++ b/ipsilon/providers/saml2/provider.py
    @@ -254,10 +254,12 @@ class ServiceProvider(ServiceProviderConfig):
             self.load_config()
     
         def get_valid_nameid(self, nip):
    -        self.debug('Requested NameId [%s]' % (nip.format,))
    -        if nip.format is None:
    +        if nip is None or nip.format is None:
    +            self.debug('No NameId requested, returning default [%s]'
    +                       % SAML2_NAMEID_MAP[self.default_nameid])
                 return SAML2_NAMEID_MAP[self.default_nameid]
             else:
    +            self.debug('Requested NameId [%s]' % (nip.format,))
                 allowed = self.allowed_nameids
                 self.debug('Allowed NameIds %s' % (repr(allowed)))
                 for nameid in allowed:
    ```

3. Install Ipsilon Client on Knox Server. Command is using "ipsilon.example.com" as the Ipsilon Server FQDN.

    ```bash
    ipsilon-client-install --saml-idp-url https://ipsilon.example.com/idp --saml-sp-name knox  --saml-auth "/gateway/knoxsso/api/v1/websso?pac4jCallback=true&client_name=SAML2Client" --saml-no-httpd 
    --saml-sp-post "/gateway/knoxsso/api/v1/websso?pac4jCallback=true&client_name=SAML2Client" --saml-sp "/gateway/knoxsso/api/v1/websso?pac4jCallback=true&client_name=SAML2Client" --saml-sp-logout="/gateway/knoxsso/api/v1/websso?pac4jCallback=true&client_name=SAML2Client" --port 8443 --saml-secure-setup=false
    ```

    It will ask for admin password on Ipsilon.

    Property Name | Description
    --------------------- | :-------------------------------:
    --saml-idp-url | URL for Ipsilon IDP Server
    --saml-sp-name | Alias to Knox Service Provider
    --saml-auth | Should match saml.serviceProviderEntityId on KnoxSSO Topology, without server:port
    --saml-no-httpd | Generate metadata and certificates local and not configure HTTPD as Service Provides (Knox will be SP)
    --saml-sp-post | Should match saml.serviceProviderEntityId on KnoxSSO Topology, without server:port
    --saml-sp | Should match saml.serviceProviderEntityId on KnoxSSO Topology, without server:port
    --saml-sp-logout | Should match saml.serviceProviderEntityId on KnoxSSO Topology, without server:port
    --port | Knox port
    --saml-secure-setup | Disable two way SSL

    This command will configure a Service Provider on Ipisilon and generate three files on the current directory: certificate.pem, certificate.key and metadata.xml.