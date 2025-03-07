.. _controller_restconf:
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

You can also configure device-groups and device-profiles as described in :ref:`the CLI tutotial <controller_cli>`, for example.

Transactions
============
Many of the controller's RPCs return a `transaction-id` that indicates that the result of
the RPC is not immediately available. Instead, it indicates that a new
transaction has been created.

.. note::
          RPCs returning a transaction-id are asynchronous

Transactions can be monitored in one of the following ways:

- Register and wait for a notification, as described in Section `notifications`_.
- Sleep/Poll and read the status of the resulting action, such as the connection status, see Section `verify connection`_.
- Sleep/Poll and read the status of the transaction using GET.

To get the status of transaction "6" using GET::

   curl -H "Accept: application/yang-data+json" -X GET
        https://localhost/restconf/data/clixon-controller:transactions/transaction=5
   HTTP/1.1 200
   {
     "clixon-controller:transaction": [
       {
         "tid": "6",
         "result": "SUCCESS", <---
         ...
       }
     ]
   }
   HTTP/1.1 200

Transactions are described in more detail in the :ref:`Transaction section<controller_transactions>`.
          
Connect
=======
If you have setup the configuration for your devices and installed the
SSH keys, you can start connecting to them. For this, you need to invoke
the ``connection-change`` RPC which starts a `device connect` transaction.

.. note::
          You need to install SSH keys before connection establishment

The connection-change RPC takes a device or device-group as input and an operation. Example::

   curl -X POST -H "Content-Type: application/yang-data+json"
       https://localhost/restconf/operations/clixon-controller:connection-change
       -d '{"clixon-controller:input":{"device":"\*","operation":"OPEN"}}'
   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"4"
      }
   }

Note that the reply contains a `transaction-id`.

Verify connection
-----------------
One way to verify a connection (apart from monitoring the transaction itself) is to wait and check the status of the connection using GET, as follows::

   curl -X GET -H "Accept: application/yang-data+json"
        https://localhost/restconf/data/clixon-controller:devices/device=openconfig1/conn-state
   HTTP/1.1 200
   {"clixon-controller:conn-state":"OPEN"}

Select devices
--------------
You can select devices in the connect RPCs as follows:

- All devices: ``device: *``
- Individual device: ``device: openconfig1``
- Device pattern: ``device: openconfig*``
- Device-groups: ``device-group: mygroup``
- Device-group pattern: ``device-group: my*``

Connect operations
------------------
The operation in the initial example is `OPEN`. The operations are:

- Establish connections to a set of devices: ``OPEN``
- Close connections: ``CLOSE``
- Close and the re-open connections: ``RECONNECT``

Example, reconnect to all devices in device-groups starting with "my*"::

   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/operations/clixon-controller:connection-change
        -d '{
              "clixon-controller:input": {
                "device-group": "my*",
                "operation": "RECONNECT"
              }
            }'
   HTTP/1.1 200

Accessing device config
=======================
When devices are open, you can get, put and push device configuration.

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

PUT
---
To edit device configuration, use PUT, POST or PATCH and then `push` the changes to devices.

With RESTCONF, modifications are written to the running datastore in the controller (local commit). Thereafter, the changes are pushed to the devices using the ``controller-commit`` RPC.

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

PUSH
----
Thereafter, push the changes to a device using the ``controller-commit`` RPC::

   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/operations/clixon-controller:controller-commit
        -d '{
              "clixon-controller:input": {
                "device": "openconfig1",
                "push": "COMMIT",
              }
            }'
   HTTP/1.1 200
   {
     "clixon-controller:output":{
       "tid":"3"
     }
   }

Again, this starts an asynchronous transaction which can be monitored with methods described in Section `transactions`_.

Notifications
=============
The controller uses notifications to get asynchronous notifications and event streams.

For example, connection establishment as described in Section
`connect`_ and commit described in Section `push`_ create
transactions. If you want to wait for such a transaction to complete,
you can register for that event stream as follows::

   curl -X GET -H "Accept: text/event-stream" -H "Cache-Control: no-cache" -H "Connection: keep-alive"
        https://localhost/streams/controller-transaction
   HTTP/2 201
   content-type: text/event-stream

   data: <notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
            <eventTime>2025-03-06T15:30:16.710209Z</eventTime>
            <controller-transaction xmlns="http://clicon.org/controller">
               <tid>4</tid>
               <username>clicon</username>
               <result>SUCCESS</result>
            </controller-transaction>
         </notification>

The `data` notification is an "SSE" / long poll event, which means
that the call blocks and waits for notifications to be received.

This means that a programmer needs to create a separate session apart
from the original RPC: One which waits for a notification, and one which
creates the transaction using an operation.

RPCs that create transactions are: ``config-pull``, ``controller-commit``, ``connection-change`` and ``device-template-apply`` (of type RPC).

Services
========
You can initate a python (PyAPI) service by editing a service and then using the ``controller-commit`` RPC with the ``action`` field set. You can also apply a service.

`Work-in-progress`

Device RPCs
===========
You can send an RPC to devices via the controller using the ``device-template-apply`` RPC.

Create template
---------------
First you create a template.

An example is the following, which is the same example as the rpc template created in the CLI as described in the :ref:`CLI tutorial <controller_cli>`::

   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/data/clixon-controller:devices
        -d '{
               "clixon-controller:rpc-template": [
                  {
                     "name": "stats",
                     "variables": {
                        "variable": [
                           {
                              "name": "MODULES"
                           }
                        ]
                     },
                     "config": {
                        "clixon-lib:stats": {
                           "modules": "${MODULES}"
                        }
                     }
                  }
               ]
            }'
   HTTP/1.1 201"

You can create the template by other means, such as CLI or NETCONF.

Send RPC
--------
The next step is to apply the template on devices resulting in a number of RPCs sent from the controller to devices.

Example::

   curl -X POST -H "Content-Type: application/yang-data+json"
        https://localhost/restconf/operations/clixon-controller:device-template-apply
        -d '{
               "clixon-controller:input": {
                  "type": "RPC",
                  "device": "openconfig*",
                  "template": "stats",
                  "variables": [
                     {
                        "variable": {
                           "name": "MODULES",
                           "value": "true"
                        }
                     }
                  ]
               }
            }'
   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"5"
      }
   }

Where a transaction id is returned.

Read result
-----------
A transaction has been created and the client needs to wait for results via a notification (see Section `notifications`_) or poll for completion of the transaction.

::

   curl -H "Accept: application/yang-data+json" -X GET
        https://localhost/restconf/data/clixon-controller:transactions/transaction=5
   HTTP/1.1 200
   {
     "clixon-controller:transaction": [
       {
         "tid": "6",
         "username": "clicon",
         "result": "SUCCESS",
         "devices": {
           "devdata": [
             {
               "name": "openconfig1",
                 "data": {
                   "global": {
                     "xmlnr": "1570",
                    "yangnr": "166357"
                 ...

Note the ``devdata`` field which returns the reply from the RPC.  That is, the reply for the ``stats`` RPC to ``openconfig1`` is::

   "data": {
     "global": {
       "xmlnr": "1570",
       "yangnr": "166357"

The ``devdata`` field may contain replies from multiple devices.
