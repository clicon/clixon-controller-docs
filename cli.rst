.. _controller_cli:
.. sectnum::
   :start: 5
   :depth: 3

************
CLI tutorial
************

This section desribes the CLI commands of the Clixon controller. A simple example is used to illustrate concepts.

Prerequisites
=============

You need a running controller and some devices. Either:

1. The docker as described in Section :ref:`Quickstart <controller_quickstart>`.
2. A host and openconfig installation as described in the :ref:`Setup tutorial <setup_tutorial>`.

General
=======

Version
-------
You can show the version either with the ``-V`` command-line option or with the CLI show version command::

  > clixon_cli -V
  Clixon version: 6.6.0
  CLIgen:         6.6.0
  Controller:     1.0.0
  Controller GIT: 40290c0
  Controller bld: 2024.02.15 13:19 by clixon on paradise

Modes
=====
The CLI has two modes: operational and configure. The top-levels are as follows::

  > clixon_cli
  cli> ?
    configure             Change to configure mode
    connection            Change connection state of one or several devices
    debug                 Debugging parts of the system
    default               Set CLI default values
    exit                  Quit
    processes             Process maintenance
    pull                  Pull config from one or multiple devices
    push                  Push config to one or multiple devices
    quit                  Quit
    save                  Save running configuration to XML file
    session               Client sessions
    shell                 System command
    show                  Show a particular state of the system

  cli> configure
  cli[/]# set ?
    devices               Device configuration
    processes             Processes configuration
    services              Placeholder for services
  cli[/]#

Devices
=======
Device configuration is separated into two domains:

1) Local information about how to access the device (meta-data)
2) Remote device configuration pulled from the device.

The user must be aware of this distinction when performing `commit` operations.

Local device configuration
--------------------------
The local device configuration contains information about how to access the device::

   device clixon-example1 {
      description "Clixon example container";
      enabled true;
      conn-type NETCONF_SSH;
      user admin;
      addr 172.17.0.3;
      yang-config VALIDATE;
   }

A user makes a local commit and thereafter explicitly connects to a locally configured device::

  # commit local
  # exit
  > connection open

Device profile
--------------
You can configure a device `profile` that applies to severaldevices. This is useful when configuring
devices of a specific vendor.

Example::

   device-profile myprofile {
      description "Clixon example container";
      conn-type NETCONF_SSH;
      user admin;
      yang-config VALIDATE;
      module-set {
         module openconfig-interfaces {
            namespace http://openconfig.net/yang/interfaces;
         }
      }
   }
   device clixon-example1 {
      device-profile myprofile;
      addr 172.17.0.3;
      enabled true;
   }
   device clixon-example2 {
      device-profile myprofile;
      addr 172.17.0.4;
      enabled true;
   }

In the example, the `myprofile` device-profile defines a set of common fields, including the locally loaded openconfig YANG. See Section :ref:`YANG <controller_yang>` for more information on loading device YANGs.

Remote device configuration
---------------------------
The remote device configuration is present under the `config` mount-point::

   device clixon-example1 {
      ...
      config {
         interfaces {
            interface eth0 {
               mtu 1500;
            }
         }
      }
   }

The remote device configuration is bound to device-specific YANG models downloaded
from the device at connection time.

Device naming
-------------
The local device name is used for local selection::

   device example1

Wild-cards (globbing) can be used to select multiple devices::

   device example*

Device groups
-------------
Device-groups can be configured and accessed as a single entity.
First, configure, a device group::

  cli# set devices mygroup example1
  cli# set devices mygroup example2
  cli# commit local

Then, use the device-group in operations::

  cli> connection open group mygroup

In the example above, both device example1 and example2 will be opened.

Note that a device-group can be:
* Hierarchical: A group may contain other groups
* Duplicates: If a device occurs twice, only one will apply
* Pattern matching: Wild-cards can be used when applying

Example::
  device-group myg*

In most commands in the following sections, device groups can be used instead of devices. In those commands, you add the keyword `group` to the command. Example::

    cli> connection open example1      # device
    cli> connection open group mygroup # device group

Connection state
----------------
Examine device connection state using the show command::

   cli> show connections
   Name                    State      Time                   Logmsg
   =======================================================================================
   example1                OPEN       2023-04-14T07:02:07
   example2                CLOSED     2023-04-14T07:08:06    Remote socket endpoint closed

Device state
------------
Device state, that is what is referred to as non-config data by YANG, is shown using::

   cli> show devices example* state
   <devdata xmlns="http://clicon.org/controller">
      <name>openconfig1</name>
      <data>
         <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
            <system xmlns="http://openconfig.net/yang/system">
               <config>
                  <hostname>openconfig1</hostname>
               </config>
               <ssh-server>
               ...

(Re)connecting
--------------
When adding and enabling one a new device (or several), the user needs to explicitly connect::

   cli> connection open <devices>
   cli> connection open group <device-group>

The "connection" command can also be used to close or reconnect devices::

   cli> connection reconnect <devices>

Device YANG
-----------
You can list which YANGs the device has using the ``show devices yang`` command::

  olof@alarik> show devices example1 yang
  example1:
  clixon-lib@2023-11-01
  clixon-restconf@2022-08-01
  ...

These YANGs are mounted specifically for this device.

Capability
----------
Use the ``show devices capability`` command to show which capabilities the device announces::

  olof@alarik> show devices example1 capability
  example1:
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.0</capability>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
    ...

The capabilities are announced as part of the initial NETCONF handshake, see `RFC 6241 <https://www.rfc-editor.org/rfc/rfc6241.html#section-8>`_ for base NETCONF capabilities.

Syncing from devices
====================
pull
----
Pull fetches the configuration from remote devices and replaces any existing device config::

   cli> pull <devices>
   cli> pull group <device-groups>

The synced configuration is saved in the controller and can be used for diffs etc.

pull merge
----------
::

   cli> pull <devices> merge

This command fetches the remote device configuration and merges with the
local device configuration. use this command with care.

Services
========
Network services are used to generate device configs.  Services are covered in more detail in the :ref:`Services tutorial <tutorial>`.

Service process
---------------
To run services, the PyAPI service process must be enabled::

  cli# set services enabled true
  cli# commit local

To view or change the status of the service daemon::

  cli> service process ?
    restart
    start
    status
    stop

Example
-------
An example service could be::

  cli> set service test 1 e* 1400

which adds MTU `1400` to all interfaces in the device config::

  interfaces {
    interface eth0{
      mtu 1400;
    }
    interface enp0s3{
      mtu 1400;
    }
  }

Service scripts are written in Python using the PyAPI, and are triggered by commit commands.

You can also trigger service scripts as follows::

  cli# apply services
  cli# apply services testA foo
  cli# apply services testA foo diff

In the first variant, all services are applied. In the second variant, only a specific service is triggered.

Created objects
---------------
The system keeps track of which device objects are created, so that they can be be removed when the service is removed. A service tags device objects with a `creator attribute` which results in a set of `created` configure objects in the controller.

The list created objects can be viewed as part of the regular configuration::

   cli> show configuration services ssh-users test1 created
   <services xmlns="http://clicon.org/controller">
      <ssh-users xmlns="urn:example:test">
         <name>test1</name>
         <created>
            <path>/devices/device[name="openconfig1"]/config/system/aaa/authentication/users/user[username="test1"]</path>
            <path>/devices/device[name="openconfig2"]/config/system/aaa/authentication/users/user[username="test1"]</path>
         </created>
      </ssh-users>
   </services>

Debugging
^^^^^^^^^
If you enable debugging (``-D app``), an entry is logged to the syslog each time the created objects change::

    Jan 22 11:24:35 totila clixon_backend[212183]: controller_edit_config:2728: Objects created in actions-db: <services xmlns="http://clicon.org/controller" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0"><ssh-users xmlns="urn:example:test"><name>test1</name><created nc:operation="merge"><path>/devices/device[name="openconfig1"]/config/system/aaa/authentication/users/user[username="test1"]</path><path>/devices/device[name="openconfig2"]/config/system/aaa/authentication/users/user[username="test1"]</path></created></ssh-users></services>

Editing
=======
Editing can be made by modifying services::

    cli# set services test 2 eth* 1500

Editing changes the controller candidate, changes can be viewed with::

   cli# show compare
        services {
   +       test 2 {
   +          name eth*;
   +          mtu 1500;
   +       }
        }

Editing devices
---------------
Device configurations can also be directly edited::

   cli# set devices device example1 config interfaces interface eth0 mtu 1500

Show and editing commands can be made on multiple devices at once using "glob" patterns::

   cli> show config xml devices device example* config interfaces interface eth0
   example1:
   <interface>
      <name>eth0</name>
      <mtu>1500</mtu>
   </interface>
   example2:
   <interface>
      <name>eth0</name>
      <mtu>1500</mtu>
   </interface>

Modifications using set, merge and delete can also be applied on multiple devices::

   cli# set devices device example* config interfaces interface eth0 mtu 9600
   cli#

Commits
=======
This section describes `remote` commit, i.e., commit operations that have to do with modifying remote device configuration. See Section `devices`_ for how to make local commits for setting up device connections.

commit diff
-----------
Assuming a service has changed as shown in the previous secion, the
`commit diff` command shows the result of running the service
scripts modifying the device configs, but with no commits actually done::

   cli# commit diff
        services {
   +       test 2 {
   +          name eth*;
   +          add 1500;
   +       }
        }
        devices {
           device example1 {
              config {
                 interfaces {
                    interface eth0 {
   -                   mtu 1400;
   +                   mtu 1500;
                    }
                 }
              }
           }
           device example33 {
              config {
                 interfaces {
                    interface eth3 {
   -                   mtu 1400;
   +                   mtu 1500;
                    }
                 }
              }
           }
        }

Commit push
-----------
The changes can now be pushed and committed to the devices::

   cli# commit push

If there are no services, changes will be pushed and committed without invoking any service handlers.

If the commit fails for any reason, the error is printed and the changes remain as prior to the commit call::

   cli# commit push
   Failed: device example1 validation failed
   Failed: device example2 out-of-sync

A non-recoverable error that requires manual intervention is shown as::

   cli# commit push
   Non-recoverable error: device example2: remote peer disconnected

To validate the configuration on the remote devices, use the following command::

   cli# validate push

If you want to rollback the current edits, use discard::

   cli# discard

One can also choose to not push the changes to the remote devices::

   cli# commit local

This is useful for setting up device connections. If a local commit is performed for remote device config, you need to make an explicit `push` as described in Section `Explicit push`_.

Limitations
-----------
The following combinations result in an error when making a remote commit:

1) No devices are present. However, it is allowed if no remote validate/commit is made. You may want to dryrun service python code for example even if no devices are present.
2) Local device fields are changed. These may potentially effect the device connection and should be made using regular netconf local commit followed by rpc connection-change, as described in Section `devices`_.
3) One of the devices is not in an OPEN state. Also in this case is it allowed if no remote valicate/commit is made, which means you can do local operations (like `commit diff`) even when devices are down.

Further, avoid doing BOTH local and remote edits simultaneously. The system detects local edits (according to (2) above) but if one instead  uses local commit, the remote edits need to be explicitly pushed

Compare and check
===============--
The "show compare" command shows the difference between candidate and running, ie not committed changes.
A variant is the following that compares with the actual remote config::

   cli> show devices <name> diff

or::

   cli> pull <name> diff

This is acheived by making a "transient" pull that does not replace the local device config.

Further, the following command checks whether devices are is out-of-sync::

   cli> show devices <name> check
   Failed: device example2 is out-of-sync

or::

   cli> pull <name> check

Out-of-sync means that a change in the remote device config has been made, such as a manual edit, since the last "pull".
You can resolve an out-of-sync state with the "pull" command.

Explicit push
=============
There are also explicit sync commands that are implicitly made in
`commit push`. Explicit pushes may be necessary if local commits are
made (eg `commit local`) which needs an explicit push. Or if a new device has been off-line::

     cli> push <devices>

Push the configuration to the devices, validate it and then revert::

     cli> push <devices> validate

Templates
=========
The controller has a simple configuration template mechanism for applying configurations to several devices at once. The template mechanism uses variable substitution.

Note there may also be templates in the Python API, these are more primitive.

A limitation is that the template itself need to be entered as XML.

.. note::
          You need to enter the template as XML

Using of a template follows the following steps:

1) Add a template using the ``load`` command and commit it
2) Apply the template using variable binding on a set of devices
3) Commit the change

Limitations
-----------
Templates are added as raw XML. The reason is that YANG-binding is not
known at the time of template creation. To know the YANG, the template
needs to be bound to some specific YANG files, or specific devices.

Since it is raw XML, there is no type-checking and any diffs (based on YANG) is limited.

.. note::
          Template XML is not type-checked and diffs are limited

Example
-------
The following example first configures a template with the formal parameters ``$NAME`` and ``$TYPE`` using the load command to paste the template config directly::

   > clixon_cli -f /usr/local/etc/clixon/controller.xml -m configure
   olof@totila[/]# load merge xml
   <config>
      <devices xmlns="http://clicon.org/controller">
         <template nc:operation="replace">
            <name>interfaces</name>
            <variables>
               <variable>
                  <name>NAME</name>
               </variable>
               <variable>
                  <name>TYPE</name>
               </variable>
            </variables>
            <config>
               <interfaces xmlns="http://openconfig.net/yang/interfaces">
                  <interface>
                     <name>${NAME}</name>
                     <config>
                        <name>${NAME}</name>
                        <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">${TYPE}</type>
                     </config>
                  </interface>
               </interfaces>
            </config>
         </template>
      </devices>
   </config>
   ^D
   olof@totila[/]# commit local
   olof@totila[/]#

The next step is to apply the configuration template: A New ``z`` interface is created on all ``openconfig`` devices::

   olof@totila[/]# apply template interfaces openconfig* variables NAME z TYPE ianaift:v35
   olof@totila[/]# show compare
               openconfig-interfaces:interfaces {
   +              interface z {
   +                 config {
   +                    name z;
   +                    type ianaift:v35;
   +                 }
   +              }
               }
               openconfig-interfaces:interfaces {
   +              interface z {
   +                 config {
   +                    name z;
   +                    type ianaift:v35;
   +                 }
   +              }
               }
   olof@totila[/]# commit
   olof@totila[/]#

RPC templates
=============
RPC templates are used to construct and send RPC:s to devices.

RPC templates are similar to configuration templates in the following way:

1) Template are defined using the ``load`` command, and then ``commit``
2) The template is applied using variable binding on a set of devices

RPC templates are `different` from configuration templates in the following way:

1) The XML format defines an RPC with input parameters instead of a configuration
2) An RPC template is applied from the operational CLI mode and no commit is made after apply

List RPC and YANG
-----------------
Before writing an RPC one can use two utility commands to list which RPC:s are defined and then study their input YANG.

You can list which RPC:s a device or a set of devices have::

   olof@alarik> show devices openconfig* rpc clixon* list
   clixon-lib:debug                 http://clicon.org/lib
   clixon-lib:ping                  http://clicon.org/lib
   clixon-lib:stats                 http://clicon.org/lib
   clixon-lib:restart-plugin        http://clicon.org/lib
   clixon-lib:process-control       http://clicon.org/lib

In the above list, all RPC:s beginning with ``clixon`` are listed from ``openconfig`` devices with their namespace.

YANG input
----------
You can also see which YANG definition an RPC has, which is convenient when writing templates. The following shows the YANG definition of the ``stats`` RPC::

   olof@alarik> show devices openconfig1 rpc clixon-lib:stats yang
   rpc stats {
      input {
         leaf modules {
             type boolean;
             mandatory false;
         }
      }
      output {
         container global{
              ...
         }
         container datastores{
               ...
         }
         container module-sets{
               ...
         }
      }
   }

where ``input`` is the model of the input parameters of the RPC and are modelled by the rpc-template, and `output` is the model of the data returned from the devices.

In the ``stats`` RPC, the input parameters is a single ``modules`` boolean leaf, while the output consists of three containers: ``global``, ``datastores``, and ``modules-sets``.

Load a template
---------------
To define a new RPC template it may be easiest to load the XML directly.
For example, define a ``clixon-lib stats`` RPC template as follows::

   > clixon_cli -f /usr/local/etc/clixon/controller.xml -m configure
   olof@totila[/]## load merge xml
   <config>
      <devices xmlns="http://clicon.org/controller">
         <rpc-template nc:operation="replace">
            <name>stats</name>
            <variables>
               <variable>
                  <name>MODULES</name>
               </variable>
            </variables>
            <config>
               <stats xmlns="http://clicon.org/lib">
                  <modules>${MODULES}</modules>
               </stats>
            </config>
         </rpc-template>
      </devices>
   </config>
   ^D
   olof@totila[/]# commit
   olof@totila[/]#

The template above contains the following components:

* A name (``stats``). This does not have to be the same as the RPC name.
* A set of formal parameters. The example contains a single ``MODULES`` parameter.
* The RPC config, must start with the RPC name '`stats`` and its namespace ``http://clicon.org/lib`` as defined by the YANG above, followed by any input variables ``<modules>${MODULES}</modules>``

Send the RPC
------------
After the RPC template is defined, it can be applied to a set of devices. In this case the template is applied on all ``openconfig`` devices and the replies are returned from ``openconfig1`` and ``openconfigs``::

   olof@totila> rpc stats openconfig* variables MODULES true
   <devdata>
      <name>openconfig1</name>
      <data>
         <global xmlns="http://clicon.org/lib">
            <xmlnr>1288</xmlnr>
            <yangnr>166303</yangnr>
         </global>
         <datastores xmlns="http://clicon.org/lib">
            <datastore>
               <name>running</name>
               <nr>113</nr>
               <size>15592</size>
            </datastore>
            ....
      <name>openconfig2</name>
      ...

RPC templates can also be used from the Python API. The following
Python code snippet shows how to use the ``device_rpc`` method to send
an RPC to a device::

   from clixon.clixon import Clixon
   cx = Clixon()

   # Send the RPC to all devices, arguments are the device name (wildcard),
   # the RPC name, and a dictionary with the input parameters
   res = cx.device_rpc("*", "stats", {"MODULES": "true"})

   for device in res:
       print(res.dumps())

Show detail
===========

The command ``show detail`` shows detailed information about configuration. The detailed output includes information about namespace, XPath etc::

   test@example> show detail ?
     devices               Device configuration
     nacm                  Parameters for NETCONF access control model.
     processes             Process configuration
     restconf              If imported, this container appears in top-level configuration.
     services              Placeholder for services.

We can then get detailed information about any of the top-level containers listed above or any item in the configuration tree below them. As an example we here show detailed information about the ``hostname`` configuration on a OpenConfig device::

   test@example> show detail devices device openconfig1 config system config hostname
   Symbol:     hostname
   Module:     openconfig-system
   File:       /usr/local/share/controller/mounts/default/openconfig-system@2024-09-24.yang
   Namespace:  http://openconfig.net/yang/system
   Prefix:     oc-sys
   XPath:      /ctrl:devices/ctrl:device[ctrl:name='openconfig1']/ctrl:config/oc-sys:system/oc-sys:config/oc-sys:hostname
   APIpath:    /clixon-controller:devices/device=openconfig1/config/openconfig-system:system/config/hostname

The XPath and APIpath can be very valuable when configuring NACM or doing RESTCONF calls.  The XPath is the full path to the configuration item in the device config, while the APIpath is the path to use when accessing the configuration via RESTCONF.

NACM
====
Clixon controller supports NACM as described in `RFC 8341 <https://www.rfc-editor.org/rfc/rfc8341.html>`_
and uses the same functionality as Clixon (`see the Clixon documentation for
more information <https://clixon-docs.readthedocs.io/en/latest/netconf.html#nacm>`_).

Device rules
------------
Rules for NACM can span over mount points and limit access to device configuration
as well as controller configuration. As an example, using an OpenConfig device it
possible to limit the access to the device hostname configuration using rules like this::

   set nacm groups group test-group
   set nacm groups group test-group user-name test
   set nacm rule-list test-rules
   set nacm rule-list test-rules group test-group
   set nacm rule-list test-rules rule test-rule
   set nacm rule-list test-rules rule test-rule path /ctrl:devices/ctrl:device[ctrl:name='*']/ctrl:config/oc-sys:system/oc-sys:config/oc-sys:hostname
   set nacm rule-list test-rules rule test-rule access-operations *
   set nacm rule-list test-rules rule test-rule action deny

With the rule above changing the hostname will result in an access-denied error::

   test@example[/]# set devices device openconfig1 config system config hostname test
   Apr 14 12:47:13.843827: clicon_rpc_edit_config: 679: Netconf error: Editing configuration: application access-denied access denied
   CLI command error

Note that in the rules the user "test" was added to the group "test-group" and
that user "test" was used to run the CLI. Also note that the path must contain
the correct namespace for the whole path.

To get the path to use in a rule it is possible to use the command "show detail"::

   test@example> show detail devices device * config system config hostname
   Symbol:     hostname
   Module:     openconfig-system
   File:       /usr/local/share/controller/mounts/default/openconfig-system@2024-09-24.yang
   Namespace:  http://openconfig.net/yang/system
   Prefix:     oc-sys
   XPath:      /ctrl:devices/ctrl:device[ctrl:name='*']/ctrl:config/oc-sys:system/oc-sys:config/oc-sys:hostname
   APIpath:    /clixon-controller:devices/device=%2A/config/openconfig-system:system/config/hostname

The XPath above is used in the NACM rule and APIpath can be used when accessing configuration
via RESTCONF.

NACM and services
-----------------
NACM rules can also be used to limit access to services. For example, the following
rule will not let the user "test" configure the service ssh-users::

   set nacm groups group test-group
   set nacm groups group test-group user-name test
   set nacm rule-list test-rules
   set nacm rule-list test-rules group test-group
   set nacm rule-list test-rules rule test-rule
   set nacm rule-list test-rules rule test-rule path /ctrl:services/ssh-users:ssh-users
   set nacm rule-list test-rules rule test-rule access-operations *
   set nacm rule-list test-rules rule test-rule action deny

When trying to do so the user will get an error message like this::

   test@example[/]# set services ssh-users test
   Apr 14 12:51:32.191560: clicon_rpc_edit_config: 679: Netconf error: Editing configuration: application access-denied access denied
   CLI command error
