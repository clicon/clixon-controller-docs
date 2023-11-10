.. _controller_cli:
.. sectnum::
   :start: 5
   :depth: 3

***
CLI
***

This section desribes the CLI commands of the Clixon controller. A simple example is used to illustrate concepts.

Modes
=====
The CLI has two modes: operational and configure. The top-levels are as follows::
   
  > clixon_cli
  cli> ?
    configure             Change to configure mode
    connection            Change connection state of one or several devices
    debug                 Debugging parts of the system
    exit                  Quit
    processes             Process maintenance 
    pull                  Pull config from one or multiple devices
    push                  Push config to one or multiple devices
    quit                  Quit
    save                  Save running configuration to XML file
    services              Services operation
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

Further, device-groups can be configured and accessed as a single entity::
  
   device-group all-examples

.. note::
          Device groups can be statically configured but not used in most operations
   
In the forthcoming sections, selecting `<devices>` means any of the methods described here.

Device state
------------
Examine device connection state using the show command::

   cli> show devices
   Name                    State      Time                   Logmsg                        
   =======================================================================================
   example1                OPEN       2023-04-14T07:02:07    
   example2                CLOSED     2023-04-14T07:08:06    Remote socket endpoint closed

There is also a detailed variant of the command with more information in XML::

   olof@zoomie> show devices detail 
   <devices xmlns="http://clicon.org/controller">
     <device>
       <name>example1</name>
       <description>Example container</description>
       <enabled>true</enabled>
       ...
  
(Re)connecting
--------------
When adding and enabling one a new device (or several), the user needs to explicitly connect::

   cli> connection <devices> connect
   
The "connection" command can also be used to close, open or reconnect devices::

   cli> connection <devices> reconnect


Syncing from devices
====================
pull
----
Pull fetches the configuration from remote devices and replaces any existing device config::

   cli> pull <devices>

The synced configuration is saved in the controller and can be used for diffs etc.


pull merge
----------
::
   
   cli> pull <devices> merge
   
This command fetches the remote device configuration and merges with the
local device configuration. use this command with care.

Services
========
Network services are used to generate device configs.

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

  cli# services reapply

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

   cli> show devices <devices> diff

This is acheived by making a "transient" pull that does not replace the local device config.

Further, the following command checks whether devices are is out-of-sync::

   cli> show devices <devices> check
   Failed: device example2 is out-of-sync

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
The controller has a simple template mechanism for applying configurations to several devices at once. The template mechanism uses variable substitution.

A limitation is that the template itself need to be entered as XML or JSON, CLI editing is not available.

.. note::
          You need to enter the template as XML

Using of a template follows the following steps:

1) Add a template using the `load` command and commit it
2) Apply the template using variable binding on a set of devices
3) Commit the change

Example
-------

The following example first configures a template with the formal parameters `$NAME` and `$TYPE`::
  
   > clixon_cli -f /usr/local/etc/clixon/controller.xml -m configure
   olof@totila[/]# load merge xml
   <config>
      <devices xmlns="http://clicon.org/controller">
         <template nc:operation="replace">
            <name>interfaces</name>
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
   olof@totila[/]# commit
   olof@totila[/]# 
      
Then, the template is applied: A ǹew `z` interface is created on all openconfig devices::

   olof@totila[/]# apply interfaces openconfig* variables NAME z TYPE ianaift:v35
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

   
