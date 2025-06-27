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
Clixon provides two separate compile-time setups for RESTCONF:

* `FCGI` / FastCGI: This solution uses a web reverse proxy such as NGINX. The reverse proxy configures all HTTP configuration. This is easier to get started.
* `Native`: which combines a HTTP and Restconf server including openssl and nghttp2. This is more advanced.

This tutorial focusses on the `FCGI` solution. The native mode is described in Section `native setup`_.

NGINX
-----
Install NGINX. Edit the location as for example as the following minimal config::

   location / {
      fastcgi_pass unix:/www-data/fastcgi_restconf.sock;
      include fastcgi_params;
   }

On many NGINX installations this can be made in ``/etc/nginx/sites-available/default``.

Restart NGINX::

   sudo systemctl restart nginx.service

Configure
---------
You need to configure clixon for FCGI::

   ./configure --with-restconf=fcgi

NGINX typically creates a `www-data` user. The following must be done prior to running:

* Create a directory for example at ``/www-data`` with the following permissions::

   drwxr-xr-x   2 www-data www-data       4096 apr  1 20:14 www-data

* Add the ``www-data`` user to the ``clicon`` group

Ensure your controller.xml configuration file has the following entries::

   <CLICON_FEATURE>clixon-restconf:allow-auth-none</CLICON_FEATURE>
   <CLICON_FEATURE>clixon-restconf:fcgi</CLICON_FEATURE>
   <CLICON_RESTCONF_DIR>/usr/local/lib/controller/restconf</CLICON_RESTCONF_DIR>

Example config
--------------
Next step is to setup RESTCONF in the datastore. FCGI configuration may look as follows::

   <restconf xmlns="http://clicon.org/restconf">
      <enable>true</enable>
      <auth-type>none</auth-type>
      <fcgi-socket>/www-data/fastcgi_restconf.sock</fcgi-socket>
      <pretty>false</pretty>
      <debug>0</debug>
      <log-destination>syslog</log-destination>
   </restconf>

Auth is set as none. If you want to use `basic auth`, you need to add
support for authentication using the ``ca_auth`` plugin callback.

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
   cli# set restconf auth-type none
   cli# set restconf fcgi-socket /www-data/fastcgi_restconf.sock
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
         <auth-type>none</auth-type>
         <fcgi-socket>/www-data/fastcgi_restconf.sock</fcgi-socket>
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

The RESTCONF daemon should be restarted automatically with the new configuration.

Verify the configuration
------------------------
Verify that the daemon is running using the CLI::

   cli> processes restconf status
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <active xmlns="http://clicon.org/lib">true</active>
      <status xmlns="http://clicon.org/lib">running</status>  <---
   </rpc-reply>

You can also verify it via RESTCONF (fields are simplified)::

   POST /restconf/operations/clixon-lib:process-control HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-lib:input": {
       "name":"restconf",
       "operation":"status"
     }
   }

A reply with a successful start is::

   HTTP/1.1 200
   {
      "clixon-lib:output": {
         "active": true,
         "description": "Clixon RESTCONF process",
         "status": "running"
      }
   }

Setup
=====
You setup the connection to one or several devices by editing the device connection data

For setup a device with IP address ``172.17.0.3`` and user ``admin`` via SSH::

   POST /restconf/data/clixon-controller:devices HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:device": {
       "name":"test",
       "enabled":"true",
       "conn-type":"NETCONF_SSH",
       "user":"admin",
       "addr":"172.17.0.3"
     }
   }

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

To get the status of transaction "6" using GET send the following request::

   POST /restconf/data/clixon-controller:transactions/transaction=6 HTTP/1.1
   Accept: application/yang-data+json

A typical reply::

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

Controller RPCs that create transactions are:

- ``config-pull``
- ``controller-commit``
- ``connection-change``
- ``device-template-apply`` (of type RPC)

Transactions are described in more detail in the :ref:`Transaction section<controller_transactions>`.

Connect
=======
If you have setup the configuration for your devices and installed the
SSH keys, you can start connecting to them. For this, you need to invoke
the ``connection-change`` RPC which starts a `device connect` transaction.

.. note::
          You need to install SSH keys before connection establishment

The connection-change RPC takes a device or device-group as input and an operation. Example::

   POST /restconf/operations/clixon-controller:connection-change HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "device":"*",
       "operation":"OPEN"
     }
   }

With the following reply containing a transaction-id::

   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"4"
      }
   }

Verify connection
-----------------
One way to verify a connection (apart from monitoring the transaction itself) is to wait and check the status of the connection using GET, as follows::

   GET /restconf/data/clixon-controller:devices/device=openconfig1/conn-state HTTP/1.1
   Accept: application/yang-data+json

   HTTP/1.1 200
   {
     "clixon-controller:conn-state":"OPEN"
   }

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

   POST /restconf/operations/clixon-controller:connection-change HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "device-group": "my*",
       "operation": "RECONNECT"
     }
   }

Accessing device config
=======================
When devices are open, you can get, put and push device configuration.

GET device config
-----------------
You can GET configuration from a single device as follows::

   GET /restconf/data/clixon-controller:devices/device=openconfig1/config HTTP/1.1
   Accept: application/yang-data+json

   HTTP/1.1 200
   {
    "clixon-controller:config": {
      "openconfig-interfaces:interfaces": {
         "interface": [
            {
               "name": "x",
               ...

You can also get more specific config::

   GET /restconf/data/clixon-controller:devices/device=openconfig1/\
       config//openconfig-interfaces:interfaces/interface=x/config/type HTTP/1.1
   Accept: application/yang-data+json

   HTTP/1.1 200
   {
      "openconfig-interfaces:type": "iana-if-type:ethernetCsmacd"
   }

PUT device config
-----------------
To edit device configuration, use PUT, POST or PATCH and then `push` the changes to devices.

With RESTCONF, modifications are written to the running datastore in the controller (local commit). Thereafter, the changes are pushed to the devices using the ``controller-commit`` RPC.

For example, change the description of an interface using PUT::

   PUT /restconf/data/clixon-controller:devices/device=openconfig1/config/\
       openconfig-interfaces:interfaces/interface=x/config HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "openconfig-interfaces:config": {
       "name": "x",
       "type": "iana-if-type:ethernetCsmacd",
       "description": "My description"
     }
   }

   HTTP/1.1 204

PUSH device config
------------------
Thereafter, push the changes to a device using the ``controller-commit`` RPC::

   POST /restconf/operations/clixon-controller:controller-commit HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "device": "openconfig1",
       "push": "COMMIT",
     }
   }

This may generate a reply as follows::

   HTTP/1.1 200
   {
     "clixon-controller:output":{
       "tid":"3"
     }
   }

Again, this starts an asynchronous transaction which can be monitored with methods described in Section `transactions`_.

PULL device config
------------------

To synchronize, that is to pull the device config to the controller, use config-pull::

   POST /restconf/operations/clixon-controller:config-pull HTTP/1.1
   Content-Type: application/yang-data+json

   {
      "clixon-controller:input": {
         "device": "openconfig1",
      }
   }

Notifications
=============
The controller uses notifications to get asynchronous notifications and event streams.

.. note::
          Notifications are not fully functional in FCGI mode

For example, connection establishment as described in Section
`connect`_ and commit described in Section `push device config`_ create
transactions. If you want to wait for such a transaction to complete,
you can register for that event stream as follows::

   GET /streams/controller-transaction HTTP/1.1
   Accept: text/event-stream
   Cache-Control: no-cache
   Connection: keep-alive

The `data` notification is an "SSE" / long poll event, which means
that the call blocks and waits for notifications to be received::

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

This means that a programmer needs to create a separate session apart
from the original RPC: One which waits for a notification, and one which
creates the transaction using an operation.

Services
========
You may wish to read the :ref:`the service tutotial <tutorial>` before reading this section.

You can initate a service typically implemented in Python(PyAPI) by editing a service configuration and then applying the service using the ``controller-commit`` RPC.

With CLI or NETCONF it is possible to edit a service in the candidate
datastore and then only trigger the services that have changed. This
is not possible in RESTCONF, since it does not use the candidate
datastore. Instead you need to explicitly set which service has
changed (or all)

Edit a service config
---------------------
First, edit a service. You can skip this part of you just want to trigger a service unconditionally.

In the following example, edit the ``bar`` instance of the ``testA`` service in module ``myyang`` (ie instance ``testA[a_name='bar']``.
::

   POST /restconf/restconf/data/clixon-controller:services HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "myyang:testA": [
       {
         "a_name": "bar",
         "params": [
           "AA"
         ]
       }
     ]
   }

This changes the service config in the running datastore of the controller. But it does not trigger a service.

Getting service diff
--------------------
To get the configuration that are going to be implemented on the devices, you need to send two RPCs
to the controller. The first one is a ``controller-commit`` and the second one is a ``datastore-diff``

The operation is::

   POST /restconf/operations/clixon-controller:controller-commit HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "push": "NONE",
       "actions": "FORCE",
       "source": "ds:candidate"
       "service-instance":"testA[a_name='bar']"
     }
   }

and then the diff can be seen by running::

   POST /restconf/operations/clixon-controller:datastore-diff HTTP/1.1
   Content-Type: application/yang-data+json

   {
      "clixon-controller:input": {
         "device": "*",
         "config-type1": "RUNNING",
         "config-type2": "ACTIONS"
      }
   }

This should be done before the service code is triggered.

Trigger service code
--------------------
To trigger a service, you need to send a ``controller-commit`` RPC to
the controller. You can do this either after editing a service, or
unconditionally, such as in a periodic process.

In the following example, you trigger the service as follows:

- It applies for all devices: ``device:*``.
- You push and commit the service result to the devices: ``push:COMMIT``. You could also just ``VALIDATE`` the service in the devices
- The service is unconditionally run: ``actions:FORCE``. This is the only option for RESTCONF.
- The datastore to work with is ``source:candidate``. This is also necessary for RESTCONF.
- Trigger the service for a specific instance: ``service-instance:testA[a_name='bar']``. You may also skip this field to trigger all services.

The corresponding operation is::

   POST /restconf/operations/clixon-controller:controller-commit HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "push": "COMMIT",
       "actions": "FORCE",
       "source": "ds:candidate"
       "service-instance":"testA[a_name='bar']"
     }
   }

Where a transaction id is returned::

   HTTP/1.1 200
   {
     "clixon-controller:output":{
       "tid":23
     }
   }

Device RPCs
===========
You can send an RPC to devices via the controller using the ``device-template-apply`` RPC.

Create template
---------------
First you create a template.

An example is the following, which is the same example as the rpc template created in the CLI as described in the :ref:`CLI tutorial <controller_cli>`::

   POST /restconf/data/clixon-controller:devices HTTP/1.1
   Content-Type: application/yang-data+json

   {
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
   }

You can create the template by other means, such as CLI or NETCONF.

Send RPC
--------
The next step is to apply the template on devices resulting in a number of RPCs sent from the controller to devices. As an alternative, you can also use `Send using XML`_.

Example::

   POST /restconf/operations/clixon-controller:device-template-apply HTTP/1.1
   Content-Type: application/yang-data+json

   {
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
   }

Where a transaction id is returned::

   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"5"
      }
   }

Read result
-----------
A transaction has been created and the client needs to wait for results via a notification (see Section `notifications`_) or poll for completion of the transaction::


   GET /restconf/data/clixon-controller:transactions/transaction=5 HTTP/1.1
   Accept: application/yang-data+json

The reply could be::

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

Inline
------
A simpler alternative is to send the template inline.

Example of ping which is also an RPC without input or output::

   POST /restconf/operations/clixon-controller:device-template-apply HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "type": "RPC",
       "device": "openconfig*",
       "inline": {
         "config": {
           "clixon-lib:ping": null
         }
       }
     }
   }

Send using XML
--------------
Instead of using JSON in the rpc-template body, you can also use XML::

   POST /restconf/operations/clixon-controller:device-template-apply HTTP/1.1
   Content-Type: application/yang-data+xml

   <input xmlns="http://clicon.org/controller">
      <type>RPC</type>
      <device>openconfig*</device>
      <inline>
         <config>
            <ping xmlns="http://clicon.org/lib"/>
         </config>
      </inline>
   </input>

Get device state
================
You can get state data from device by using an RPC template as a workaround for not supplying it with a top-level `GET`.

Device state using XML
----------------------
Example, get the ssh state of all openconfig devices::

   POST /restconf/operations/clixon-controller:device-template-apply HTTP/1.1
   Content-Type: application/yang-data+xml

   <input xmlns="urn:example:clixon-controller">
      <type>RPC</type>
      <device>openconfig*</device>
      <inline>
         <config>
            <get xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
               <filter type="xpath" select="/oc-sys:system/oc-sys:ssh-server"
                       xmlns:oc-sys="http://openconfig.net/yang/system" />
            </get>
         </config>
      </inline>
   </input>

   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"6"
      }
   }

where the `filter` statement selects the ``system/ssh-server`` state.

Polling for result::

   GET /restconf/data/clixon-controller:transactions/transaction=6 HTTP/1.1
   Accept: application/yang-data+json

A result could be::

   HTTP/2 200
   content-type: application/yang-data+json

   {
     "clixon-controller:transaction": [
      {
        "tid": "8",
        "username": "anonymous",
        "result": "SUCCESS",
        "devices": {
          "devdata": [
           {
            "name": "openconfig1",
            "data": {
              "data": {
                "system": {
                  "ssh-server": {
                    "state": {
                      "enable": "true",
                      "protocol-version": "V2"
                    }
                  }
                }
               }
             }
           },
           ...

where the result of the first matchong device (``openconfig1``) is shown.

Device state using JSON
-----------------------
At this time it is not possible to get a subset of subset of state data for JSON, ie an XPath selection, the whole state data is returned.

.. note::
          You cannot get individual elements via JSON, just ALL device state

Example, get all state of all "openconfig*" devices::

   POST /restconf/operations/clixon-controller:device-template-apply HTTP/1.1
   Content-Type: application/yang-data+json

   {
     "clixon-controller:input": {
       "type":"RPC",
       "device":"openconfig*",
       "inline": {
         "config": {
           "ietf-netconf:get":null
         }
       }
     }
   }

   HTTP/1.1 200
   {
      "clixon-controller:output":{
         "tid":"6"
      }
   }

Polling for result::

   GET /restconf/data/clixon-controller:transactions/transaction=6 HTTP/1.1
   Accept: application/yang-data+json

where the result would be the complete state of all matching devices.

Native setup
============
Native mode is more complex to setup and provides many different configurations for RESTCONF. The controller supports the following:

1. Native TLS and http in the RESTCONF daemon. No reverse proxy is needed.
2. HTTP/1.1 and HTTP/2
3. Basic and TLS/SSL client cert authentication
4. Datastore configuration, not in configuration file

Configuration
-------------
You need to configure clixon for native::

   ./configure --with-restconf=native

Example config
--------------
A typical RESTCONF native configuration may look as follows::

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

In the config where a TLS on port 443 on 192.168.32.1 is configured using client-certs placed in the ``etc/pki/tls`` directory.

Alternatively, you may use `basic auth`, but then you need to add
support for authentication using the ``ca_auth`` plugin callback.

For testing purposes, ``none`` can be used as auth-type.
