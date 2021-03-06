=====================================================
Troubleshooting Chef Automate
=====================================================
`[edit on GitHub] <https://github.com/chef/chef-web-docs/blob/master/chef_master/source/troubleshooting_chef_automate.rst>`__

.. tag chef_automate_mark

.. image:: ../../images/chef_automate_full.png
   :width: 40px
   :height: 17px

.. end_tag

The following are issues that you may encounter during the setup and usage of Chef Automate.

Preflight Check
=====================================================

This is a list of possible error codes and remediation steps you might see when running the ``preflight-check`` command before setting up your Chef Automate server:

*   PF01: The system does not have this given directory or is not writable.

    *   Please create the directory or make sure it is writable.

*   PF02: The system umask is not 0022.

    *   Please change it to 0022.

*   PF03: System has less than 4 CPU cores.

    *   Please increase CPU cores to at least 4.

*   PF04: System has less than 80GB disk space at `/var`.

    *   Please increase free disk space at `/var` to at least 80GB.

*   PF05: System has less than 16GB of free memory.

    *   Please increase free memory to at least 16GB.

*   PF06: Another process is using this port on the system.

    *   Please free this port on the system.

*   PF07: System does not have this software installed.

    *   Please install this software via the given command.


Build nodes/Runners
=====================================================

The following are possible issues you might run into when using build nodes and/or runners.

Issue: Waiting for builder.
-----------------------------------------------------
If "waiting for builder" occurs in the log output on a new Chef Automate setup with no existing build nodes, then the Chef Automate server and Chef server are not communicating. To establish communication, try restarting Chef Automate's main service with ``automate-ctl restart delivery``.

If "waiting for builder" occurs in the log output on a Chef Automate setup with existing build nodes, then it indicates incorrect mapping between between v1/v2 build job type and the available builder/runner resources in a Chef Automate cache. Check your project's ``.delivery/config.json`` to confirm that it correctly represents the use of builders/runners, adjust this if necessary, and restart Automate's main service with ``automate-ctl restart delivery``.

If your Chef Automate system has builders(push jobs), then your projects should have the following configuration in .delivery/config.json :

   .. code-block:: none

       "job_dispatch": {
         "version": "v1"
       },

If your Chef Automate system has runners, then your projects should have the following configuration in .delivery/config.json

   .. code-block:: none

       "job_dispatch": {
         "version": "v2"
       },


If the ``.delivery/config.json`` is correct, but jobs are not kicking off, then the best thing to do is restart Automate's main service with ``automate-ctl restart delivery``. After restarting the service, queued change jobs should start being processed by the available resources for that job type.

Issue: No build nodes/runners available.
-----------------------------------------------------

If you see "no build nodes available" in your log output, then you need to set up build nodes.
If you have set up build nodes and are still seeing this error, then you need to check if the build nodes registered with the chef server correctly.  In this case, a correct registration is something that matches your build node query.

By default, Chef Automate build nodes/runners generated by ``automate-ctl install-build-node`` or ``automate-ctl install-runner``, which are respectively tagged as ``delivery-build-node`` and ``delivery-job-runner``. If your delivery.rb contains a custom search query (``delivery['default_search']`` is set), try appending ``" OR tags:delivery-build-node"`` or ``" OR tags:delivery-job-runner"`` to your query.

At a minimum, the build-node and runner configuration includes the following:

If your Chef Automate system has builders(push jobs), then your projects should have the following configuration in .delivery/config.json :

   .. code-block:: none

       "job_dispatch": {
         "version": "v1"
       },

If your Chef Automate system has runners, then your projects should have the following configuration in .delivery/config.json

   .. code-block:: none

       "job_dispatch": {
         "version": "v2"
       },


If you are trying debugging a specific build node or runner and need to ensure that one is available for your projects,
then modify the build-nodes or job_dispatch default search for your project as described in :doc:`Configure a Project </config_json_delivery>`.

SAML Authentication
=======================================================

When setting up SAML authentication, you might run into the following issues where you cannot sign in with SAML.

Issue: The browser shows a blank page.
-----------------------------------------------------

If both of these conditions are true:

* The URL of the blank page is ``https://<yourChef AutomateDomain>/api/v0/e/<enterprise>/saml/auth/<my-saml-name>``
* The logs show ``[error] Ranch listener http terminated in auth_hand_saml_auth:handle/2 with reason: no match of right hand value false in base64:decode_binary/2 line 212``

then the SAML IdP certificate stored in the database needs to be base64-encoded.

You can verify that a certificate is correctly copied by doing the following:

#. Save the certificate to a file (e.g. `CERT`).
#. In the command line, run ``base64 -D CERT | openssl x509 -inform DER -text -noout``.

   The output should be the certificate information, for example

   .. code-block:: none

      Certificate:
         Data:
            Version: 3 (0x2)
            Serial Number:
                  01:4b:41:db:a2:9c
            Signature Algorithm: sha1WithRSAEncryption
            Issuer: C=US, ST=California, L=San Francisco, O=Okta, OU=SSOProvider, CN=getchef/emailAddress=info@okta.com
      ...

      .. note:: The `base64` CLI tool is not as strict in decoding Base64 as Erlang is.

If the output from the above commands displays the certificate info, but you still get the error pattern, then try running your certificate through Erlang:

#. Open an Erlang shell: ``erl``.
#. Type ``{ok, Content} = file:read_file(Path).`` to read the file (note the period at the end).
#. Type ``base64:decode(Content).`` to try decoding the base64-encoded certificate.

If the certificate can be decoded, you should see something like:

.. code-block:: none

   > base64:decode(Content).
   <<48,130,3,158,48,130,2,134,160,3,2,1,2,2,6,1,75,65,219,
     162,156,48,13,6,9,42,134,72,134,...>>

and if it can't be decoded:

.. code-block:: none

   > base64:decode(Content).
   ** exception error: no match of right hand side value false
       in function  base64:decode_binary/2 (base64.erl, line 212)

Issue: The browser shows the login UI with "SAML login failed!"
-----------------------------------------------------------------

Case #1
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

If you see this error and the logs show ``Invalid assertion {assertion,{error,cert_not_accepted}}``, then the stored certificate is base64-encoded, but is the incorrect certificate for the IdP for signing the assertion response.

To find the correct certificate, you can examine the assertions given by the IdP on successful login:

#. Open Chrome's "Developer Tools" (Alt+Cmd+i on macOS) > Network (4th tab).
#. Select `Preserve Log` (2nd row) and `All` (3rd row).
#. Try logging in via SAML again.
#. Find the request to `consume` (Name column).
#. In the`Header` tab, scroll down to `Form Data` and copy the `SAMLResponse` data.
#. Go to https://www.samltool.com/decode.php and paste the SAMLResponse, click `decode and inflate XML`.
#. Compare the certificate in the XML document (``ds:X509Certificate`` or a similar tag) to the certificate stored in the SAML Setup page.

Case #2
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

If you see this error and the logs show ``[error] Invalid assertion bad_recipient``, then the "Assertion Consumption Service" (ACS)
endpoint configured with the IdP is incorrect.

A configuration mismatch of this kind most likely breaks the interaction completely. Seeing this error hints at a minor mismatch -- most likely concerning the `api_proto` setting.

Follow the steps provided in Case #1 to examine the assertions returned from the IdP and verify that the recipient of the assertion response matches Chef Automate's saml/consume endpoint:

.. code-block:: none

   <?xml version="1.0" encoding="UTF-8"?>
     <saml2p:Response
        xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol"
        Destination="http://<yourChef AutomateDomain>/api/v0/e/cd/saml/consume" <<< THIS NEEDS TO MATCH
        ID="id106938446989890821534691506"
        InResponseTo="_209b55372ca56aee1457a2f6a5eced8e"
        IssueInstant="2016-06-13T12:03:04.758Z"
        Version="2.0"
        xmlns:xs="http://www.w3.org/2001/XMLSchema">

Case #3
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

If you see this error and the logs show ``[error] Invalid assertion bad_in_response_to``, then the response does not match a request.

.. code-block:: none

   <?xml version="1.0" encoding="UTF-8"?>
     <saml2p:Response
        xmlns:saml2p="urn:oasis:names:tc:SAML:2.0:protocol"
        Destination="http://<delivery>/api/v0/e/cd/saml/consume"
        ID="id106938446989890821534691506"
        InResponseTo="_209b55372ca56aee1457a2f6a5eced8e" <<< THIS NEEDS TO MATCH
        IssueInstant="2016-06-13T12:03:04.758Z"
        Version="2.0"
        xmlns:xs="http://www.w3.org/2001/XMLSchema">

This can happen when either the IdP is not compliant to the SAML specs, or when the assertion is too late, that is, when the initiation of the SAML login process
(the redirect to your IdP) has been longer than 15 minutes.

Issue: The browser shows the login UI with "Invalid user, login failed!"
-------------------------------------------------------------------------

Chef Automate does not have a user-record for the user information from the SAML assertion.
This can be triggered by either:

* Initiating SAML authentication when trying to log in by entering a username of a Chef Automate user with authentication type SAML.
* When redirected to the SAML IdP, authenticating as a different user (not known to Chef Automate).

This can also indicate a change in NameId settings.

Visibility
====================================================================

The following is an issue you might run into when using the visibility capabilities in Chef Automate.

Issue: Data does not show up in Chef Automate UI.
------------------------------------------------------------------------------------

.. tag chef_automate_visibility_no_data_troubleshoot

If an organization does not have any nodes associated with it, it does not show up in the **Nodes** section of the Chef Automate UI.
This is also true for roles, cookbooks, recipes, attributes, resources, node names, and environments. Only those items
that have a node associated with them will appear in the UI. Chef Automate has all the data for all of these, but does
not highlight them in the UI. This is designed to keep the UI focused on the nodes in your cluster.

.. end_tag
