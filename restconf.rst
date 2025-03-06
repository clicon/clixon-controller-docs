.. _controller_cli:
.. sectnum::
   :start: 7
   :depth: 3

********
RESTCONF
********

This section describes how to use RESTCONF with the Clixon controller.

Please also consult the RESTCONF section in the `Clixon user manual <https://clixon-docs.readthedocs.io>`_.

Configuration
=============

Clixon provides many different configurations for RESTCONF. The controller only supports the following:

1. Native TLS and http in the RESTCONF daemon. No reverse proxy is needed.
2. HTTP/1.1 and HTTP/2
3. Basic and TLS/SSL client cert authentication
4. Datastore configuration, not in configuration file

Example config
--------------
A typical RESTCONF configuration may look as follows, where a TLS on port 443 on 192.168.32.1 is configured using client-certs placed in the ``etc/pki/tls`` directory::

   <restconf xmlns="http://clicon.org/restconf">
      <enable>true</enable>
      <auth-type>client-certificate</auth-type>
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

Alternatively, you may use `basic auth`

You should modify the configuration above to suit you needs,
thereafter install it in the Clixon datastore using one of the methods
described in the next section.

Install configuration
=====================
You install the RESTCONF configuration by adding it to the datastore in one of the following methods.

Install using CLI
-----------------
Enter the CLI and edit the RESTCONF configuration and commit it::

   # clixon_cli
   cli> configure
   cli# set restconf enable true
   cli# set restconf server-cert-path /var/tmp/./test-restconf.sh/certs/clixon-server-crt.pem
   cli# set restconf server-key-path /var/tmp/./test-restconf.sh/certs/clixon-server-key.pem
   cli# set restconf server-ca-cert-path /var/tmp/./test-restconf.sh/certs/clixon-ca-crt.pem
   cli# set restconf socket default 0.0.0.0 443
   cli# set restconf socket default 0.0.0.0 443 ssl true
   cli# commit local
   cli#

The commit command should (re)start the RESTCONF daemon with the new configuration. To verify that the RESTCONF is running, see Section `Verify the configuration`_.
   
Install in datastore
--------------------
If you use a `startup-db` or `running-db` you can directly edit the datastore by adding the restconf config and restart the ``clixon_backend``.

Add the restconf config to the datastore, such as ``/usr/local/var/controller/startup.d/0.xml`` as follows::

   <config>
      ...
      <restconf xmlns="http://clicon.org/restconf">
         <enable>true</enable>
         ...
      </restconf>
   </config>

Then restart ``clixon_backend``, typically using systemd::

   sudo systemctl restart clixon-controller.service

Install using NETCONF
---------------------

Send a NETCONF edit-config message to modify the restconf configuration, and then commit it::

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

The RESTCONF daemon should be restarted with the new configuration.
   
Verify the configuration
------------------------
Verify that the daemon is running using the CLI::

   cli> processes restconf status
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <active xmlns="http://clicon.org/lib">true</active>
      <status xmlns="http://clicon.org/lib">running</status>  <---
   </rpc-reply>

You can also verify it via RESTCONF (using curl as a tool)::
  
   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/operations/clixon-lib:process-control
        -d '{"clixon-lib:input":{"name":"restconf","operation":"status"}}'
   HTTP/1.1 200
   {
      "clixon-lib:output": {
         "active": true,
         "description": "Clixon RESTCONF process",
         "status": "running",  <---
      }
   }

Setup
=====
You setup the connection to one or several devices by editing the device connection data

For example, using curl, setup a device with IP address ``172.17.0.3`` and user ``admin`` via SSH::

   curl -X POST -H "Content-Type: application/yang-data+json"
     https://localhost/restconf/data/clixon-controller:devices
     -d '{"clixon-controller:device":
            {"name":"test",
             "enabled":"true",
             "conn-type":"NETCONF_SSH",
             "user":"admin",
             "addr":"172.17.0.3"
            }
         }'
   HTTP/1.1 201

Connect
=======

If you have setup the configuration to your devices and installed the
SSH keys, you can establish connections to devices. For this, you need to invoke
the `connection-change` RPC.

.. note::
          You need to install SSH keys before connection establishment
          
Example::

   curl -X POST -H "Content-Type: application/yang-data+json"
       https://localhost/restconf/operations/clixon-controller:connection-change
       -d '{"clixon-controller:input":{"device":"\*","operation":"OPEN"}}'
   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"4"
      }
   }

In the example, all devices are selected, and the operation is `OPEN`. You can also close, or reconnect.

To instead make a "glob" pattern matching a set of device-groups::

   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/operations/clixon-controller:connection-change
        -d '{"clixon-controller:input":{"device-group":"openconfig*","operation":"OPEN"}}'
   HTTP/1.1 200

The return value of the connect operation is a `transaction-id`. Connection establishment is asynchronous and can be monitored by a notification, which is further described in Section `notifications`_.

Another alternative is to wait and check the status of the connection using GET, as follows::

   curl -X GET -H "Accept: application/yang-data+json"
        https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/conn-state
   HTTP/1.1 200
   {"clixon-controller:conn-state":"OPEN"}

Accessing device config
=======================

When devices are open, you can retreive and edit device configuration.

GET
---
You can GET configuration from a single device as follows::

   curl -H "Accept: application/yang-data+xml" -X GET
        https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/config
   HTTP/1.1 200
   {
    "clixon-controller:config": {
      "openconfig-interfaces:interfaces": {
         "interface": [
            {
               "name": "x",
               ...

You can also get more specific config::

   curl -H "Accept: application/yang-data+xml" -X GET
        https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/config//openconfig-interfaces:interfaces/interface=x/config/type
   HTTP/1.1 200
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
   HTTP/1.1 204

Notifications
=============


Device RPCs
===========

Services
========

