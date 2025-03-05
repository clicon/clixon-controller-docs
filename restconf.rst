.. _controller_cli:
.. sectnum::
   :start: 7
   :depth: 3

********
RESTCONF
********

This section desribes how to use RESTCONF with the Clixon controller.


Configuration
=============

Clixon provides many different configurations for RESTCONF. The controller only supports one, as follows:

1. Native mode, that is, TLS and http is native in the RESTCONF daemon. No reverse proxy is needed.
2. HTTP/1.1 and HTTP/2 is supported
3. Only basic and SSL client cert authentication is supported
4. The RESTCONF configuration is part of the `datastore`. That is, not in the configuration file

Please also consult the RESTCONF section in the `Clixon user manual <https://clixon-docs.readthedocs.io>`_.

Example config
--------------
A typical configuration using client certs looks something like this::

   <restconf xmlns="http://clicon.org/restconf">
      <enable>true</enable>
      <debug>0</debug>
      <timeout>0</timeout>
      <auth-type>client-certificate</auth-type>
      <pretty>false</pretty>
      <server-cert-path>/etc/pki/tls/certs/clixon-server-crt.pem</server-cert-path>
      <server-key-path>/etc/pki/tls/private/clixon-server-key.pem</server-key-path>
      <server-ca-cert-path>/etc/pki/tls/CA/clixon-ca-crt.pem</server-ca-cert-path>
      <socket>
         <namespace>default</namespace>
         <address>192.168.32.1</address>
         <port>443</port>
         <ssl>true</ssl>
      </socket>
   </restconf>

The config above listen to TLS on port 443 on 192.168.32.1 using client-certs.

Modify the configuration above to suit you needs, place it in the datastore using one of the methods described in the next section.

Starting
========
You start the RESTCONF daemon by adding the configuration to the datastore in one of the following methods.

CLI
---
Enter the CLI and edit the restconf configuration and commit it::

   # clixon_cli
   cli> configure
   cli# set restconf enable true
   cli# set restconf server-cert-path /var/tmp/./test-restconf.sh/certs/clixon-server-crt.pem
   cli# set restconf server-key-path /var/tmp/./test-restconf.sh/certs/clixon-server-key.pem
   cli# set restconf server-ca-cert-path /var/tmp/./test-restconf.sh/certs/clixon-ca-crt.pem
   cli# set restconf socket default 0.0.0.0 443
   cli# set restconf timeout 10
   cli# set restconf socket default 0.0.0.0 443 ssl true
   cli# commit local
   cli#

Datastore
---------
If you use a `startup-db` or `running-db` you can directly edit the datastore by adding the restconf config and restart the ``clixon_backend``.

Add the restconf config to the datastore, such as ``/usr/local/var/controller/startup.d/0.xml`` as follows::

   <config>
      ...
      <restconf xmlns="http://clicon.org/restconf">
         <enable>true</enable>
         ...
      </restconf>
   </config>

Then restart ``clixon_backend``

NETCONF
-------

Send a NETCONF message to the controller, something like::

   <?xml version="1.0" encoding="UTF-8"?>
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <capabilities>
         <capability>urn:ietf:params:netconf:base:1.0</capability>
      </capabilities>
   </hello>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="42">
      <target>
         <candidate/>
      </target>
      <config>
         <restconf xmlns="http://clicon.org/restconf">
            <enable>true</enable>
            ...
         </restconf>
      </config>
   </rpc>]]>]]>
   <rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="42">
      <commit/>
   </rpc>]]>]]>


Operation
=========

Verify
------
Verify that the daemon is running in the CLI::

   cli> processes restconf status
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <active xmlns="http://clicon.org/lib">true</active>
      <description xmlns="http://clicon.org/lib">Clixon RESTCONF process</description>
      <command xmlns="http://clicon.org/lib">/usr/local/sbin/clixon_restconf -f /usr/local/etc/clixon/controller.xml -E /var/tmp/./test-restconf.sh/confdir -D 0 -l s</command>
      <status xmlns="http://clicon.org/lib">running</status>
      <starttime xmlns="http://clicon.org/lib">2025-03-03T16:01:53.464749Z</starttime>
      <pid xmlns="http://clicon.org/lib">29238</pid>
   </rpc-reply>

Note the status: running field.

Connect
=======
You can connect to devices using curl (or other similar tool) to invoke the `connection-change` operationas. For example::

   curl -X POST -H "Content-Type: application/yang-data+json" https://localhost/restconf/operations/clixon-controller:connection-change -d '{"clixon-controller:input":{"device":"*","operation":"OPEN"}}'
   {
      "clixon-controller:output":{
         "tid":"4"
      }
   }

In the example, all devices are selected, and the operation is `OPEN`. You can also close, or reconnect.

The return value of the connect operation is a `transaction-id`. Connection establishment is asynchronous and can be monitored by a notification, which is further described in Section `notifications`_.

Another alternative is to wait and check the status of the connection using GET, as follows::

   curl -X GET -H "Accept: application/yang-data+json" https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/conn-state
   {"clixon-controller:conn-state":"OPEN"}

Accessing device config
=======================

When devices are open, you can retreive and edit device configuration.

GET
---
You can GET configuration from a single device as follows::

   curl -H "Accept: application/yang-data+xml" -X GET https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/config
   {
    "clixon-controller:config": {
      "openconfig-interfaces:interfaces": {
         "interface": [
            {
               "name": "x",
               ...

You can also get more specific config::

   curl -H "Accept: application/yang-data+xml" -X GET https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/config//openconfig-interfaces:interfaces/interface=x/config/type
   {
      "openconfig-interfaces:type": "iana-if-type:ethernetCsmacd"
   }

Edit
----
To edit device configuration, use PUT, POST or PATCH.
For example, change the description of an interface using PUT::

   curl -X PUT -H "Content-Type: application/yang-data+json"
      https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/config/openconfig-interfaces:interfaces/interface=x/config
      -d '{
           "openconfig-interfaces:config": {
             "name": "x",
             "type": "iana-if-type:ethernetCsmacd",
             "description": "My description"
             }
          }'

RPCs
====


Notifications
=============

Services
========

