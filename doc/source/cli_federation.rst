=================================================
Using openstackclient with user federation on OTC
=================================================

Configuring your Openstack Client is mostly done by using the ``clouds.yaml`` file under ``~/.config/openstack/clouds.yaml``

You can see the basic settings `here <https://docs.otc.t-systems.com/docsportal/configuration.html>`_

Open Telekom Cloud also allows you to federate your user management to an `external Identity Provider <https://docs.otc.t-systems.com/identity-access-management/umn/user_guide/federated_identity_authentication/introduction.html>`_. 
For this, Open Telekom Cloud supports `SAML <https://docs.otc.t-systems.com/identity-access-management/umn/user_guide/federated_identity_authentication/saml-based_federated_identity_authentication/index.html>`_ and `OpenID Connect <https://docs.otc.t-systems.com/identity-access-management/umn/user_guide/federated_identity_authentication/openid_connect_based_federated_identity_authentication/index.html>`_ Identity Providers.
Once you have setup your idp in IAM, you can login into the webconsole via SAML or OIDC. 

The openstack client must first be configured to make use of the user federation.




OpenID connect
==============


Prepare 2 files under ``~/.config/openstack/``, your ``clouds.yaml`` and your ``secure.yaml`` for keeping the credentials seperate

**clouds.yaml**

.. code-block:: yaml

  clouds:
    tenant_with_oidc:
      auth_url: "https://iam.eu-de.otc.t-systems.com:443/v3"
      identity_api_version: "3"
      auth:
        project_id: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
      auth_type: 'v3oidcpassword'
      username: 'user1'
      identity_provider: 'myOpenIDProvider' #correlates to the name you gave your idp in the IAM settings
      protocol: 'oidc'
      client_id: 'opentelekomcloud' #you'll need to configure the client on your idp, in this case, OTC IAM would be the client, as it consumes the provided identities.
      discovery_endpoint: 'https://youridp.com/openid-configuration' #Your idps OIDC discovery endpoint.
      access_token_type: 'id_token'	




**secure.yaml**

.. code-block:: yaml

  clouds:
    tenant_with_oidc:
      password: 'Test1234'
      client_secret: 'zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz'




SAML
====

**clouds.yaml**

.. code-block:: yaml

  clouds:	
    tenant_with_saml:
      auth_url: "https://iam.eu-de.otc.t-systems.com:443/v3"
      identity_api_version: "3"
      auth:
        project_id: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
      auth_type: 'v3samlpassword'
      username: 'user1'
      identity_provider: 'mySAMLIDProvider'  #correlates to the name you gave your idp in the IAM settings
      protocol: 'saml'
      identity_provider_url: 'https://youridp.com/samlassertionendpoint'


**secure.yaml**

.. code-block:: yaml

  clouds:	
    tenant_with_saml:
      password: 'Test54321!'




.. code-block:: yaml

  clouds:
    otcfirstproject:
      profile: otc
      auth:
        username: '<USER_NAME>'


Please note, the SAML IDP musst support the so called `ECP-Flow <http://docs.oasis-open.org/security/saml/Post2.0/saml-ecp/v2.0/saml-ecp-v2.0.html>`_.
In Keycloak this needs to be enabled explicitly. AWS SSO (AWS IAM Identity Center) does not support ECP at all, so using the openstackclient in this combination would not be possible.
Check the documentation of your idp to see if it supports SAML ECP
