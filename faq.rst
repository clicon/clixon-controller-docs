.. _controller_install:
.. sectnum::
   :start: 15
   :depth: 3

********************************
FAQ (Frequently Asked Questions)
********************************

Clixon Controller FAQ
======================

  * `What is the Clixon controller?`_
  * `Why should I use a network controller?`_
  * `What is NETCONF/YANG?`_
  * `What about other methods, such as SNMP or Ansible?`_
  * `How does the controller differ from Clixon?`_
  * `What about other protocols?`_
  * `My devices are CLOSED`_
  * `How to configure JunOS and the Clixon controller?`_
  * `How do I add a device in Clixon?`_
  * `My device symbols do not appear in the CLI?`_
  * `My device does not announce models?`_
  * `What about YANG features?`_
  * `What about the directory structure?`_
  * `Candidate is locked`_
  * `Can Openconfig and IETF YANG co-exist?`_

What is the Clixon controller?
------------------------------
The Clixon controller is an open-source controller for network devices with a NETCONF/YANG API.

Its aim is to provide centralized automation of device operation using an interactive CLI and a Python engine over a network.

The controller sets up connections to devices, and controls them by monitoring their status and automate their configuration.

It supports multiple devices with different YANG models from different vendors.

Why should I use a network controller?
--------------------------------------
A network controller provides automation and a more abstract way of configuring a set of devices, in comparison with manual maintainence.

What is NETCONF/YANG?
---------------------
NETCONF is a network management protocol providing a transactional semantics (commit/rollback).
YANG is a data modeling language for the definition of data sent over (for example) NETCONF.
NETCONF and YANG are IETF standards and a growing number of devices provide APIs.

What about other methods, such as SNMP or Ansible?
--------------------------------------------------
There are many ways to manage devices. You choose.

How does the controller differ from Clixon?
-------------------------------------------
Clixon itself is mainly used for end-systems, mostly network devices
(such as firewalls, routers, switches), in some "embedded" form.
However, Clixon can also be used as a platform for applications where
its semantics is implemented by plugins.  The controller is such an
application: I.e., it is a Clixon application. All controller
semantics is implemented by plugins: backend plugins for communicating
with remote devices and CLI plugins for the specialized CLI interface.

What about other protocols?
---------------------------
The clixon controller only interfaces with devices using NETCONF, not
other protocols are supported (or CLI).  The controller itself
supports NETCONF, RESTCONF and CLI.

My devices are CLOSED
---------------------
If a device does not come up and shows something like::

  cli>show device
  Name                    State      Time                   Logmsg
  =======================================================================================
  clixon-example1         CLOSED     2023-05-25T11:12:29    Closed by device

The controller may be unable to login to the device for one of the following reasons:

   * The device has not netconf SSH subsystem enabled
   * The controllers public SSH key is not installed on the device
   * The device host key is not installed in the controllers `known_hosts`

The controller requires its public key to be installed on the devices and performs strict checking of host keys to avoid man-in-the-middle attacks. You need to ensure that the public key the controller uses is installed on the devices, and that the known_hosts file of the controller contains entries for the devices.

How to configure JunOS and the Clixon controller?
-------------------------------------------------
JunOS must be configured with SSH-keys and a few other settings before being used with Clixon. The SSH-key belongs to the user which clixon_backend run as. The rfc-compliant option for the netconf server must also be set::

  root@junos> show configuration
  ## Last commit: 2023-05-22 13:04:40 UTC by admin
  version 20220909.043510_builder.r1282894;
  system {
      login {
          user admin {
              uid 2000;
              class super-user;
              authentication {
                ssh-rsa "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDF7BB+hkfqtLiwSvPNte72vQSzeF/KRtAEQywJtqrBAiRJBalZ30EyTMwXydPROHI5VBcm6hN28N89HtEBmKcrg8kU7qVLmmrBOJKYWI1aAWTgfwrPbnSOuo4sRu/jUClSHryOidEtUoh+SJ30X1yvm+S2rP0TM8W5URk0KqLvr4c/m1ejePhpg4BElicFwG6ogZYRWPAJZcygXkGil6N2SMJqFuPYC+IWnyh1l9t3C1wg3j1ldcbvagKSp1sH8zywPCfvly14qIHn814Y1ojgI+z27/TG2Y+svfQaRs6uLbCxy98+BMo2OqFQ1qSkzS5CyEis5tdZR3WW917aaEOJvxs5VVIXXb5RuB925z/rM/DwlSXKzefAzpj0hsrY365Gcm0mt/YfRv0hVAa0dOJloYnZwy7ZxcQKaEpDarPLlXhcb13oEGVFj0iQjAgdXpECk40MFXe//EAJyf4sChOoZyd6MNBlSTTOSLyM4vorEnmzFl1WeJze5bERFOsHjUM="; ## SECRET-DATA
              }
          }
      }
      services {
          ssh;
          netconf {
              ssh;
              rfc-compliant;
              yang-compliant;
          }
      }
  }

How do I add a device in Clixon?
--------------------------------
The device should be configured to use the same user as in the configuration above, using the CLI::

  set devices device test enabled true
  set devices device test conn-type NETCONF_SSH
  set devices device test user admin
  set devices device test addr 1.2.3.4
  commit local

Thereafter the device must be explicitly connected::

  connection open

My device does not announce models?
-----------------------------------
The main mechanism in the controller to get YANGs from devices is the RFC6022 `get-schema` mechanism.

If a device does not support this mechanism, the following error appears::

  Netconf monitoring capability not announced in hello protocol and no local models found

If the device does not announce its models in this way, you can declare a local `module-set` which is loaded instead::

  cli# set devices device-profile myprofile module-set module mymodule namespace ...

A related error is that the device announex models but have an unrecognized `location`, which clixon does not support. The error message is::

  Module: mymodule: Unsupported location:http://...

The workaround for this is to download the yang models out-of-band and
store them somewhere accessible by that domain.

See the user-guide for details: https://clixon-controller-docs.readthedocs.io/en/latest/yang.html

My device symbols do not appear in the CLI?
-------------------------------------------
Connections are open but when you try to edit the device config, there are no symbols::

   controller[/]# set devices device mydev config ?
   controller[/]#

It may be that your `autocli.xml` file needs editing. Suppose your YANG files start with prefix `myyang-`. Edit the existing `autocli.xml` or add a new file `/usr/local/etc/controller/autocli-myang.xml` as follows::

  <?xml version="1.0" encoding="utf-8"?>
  <clixon-config xmlns="http://clicon.org/config">
    <autocli>
       <module-default>false</module-default>
       <list-keyword-default>kw-nokey</list-keyword-default>
       <treeref-state-default>true</treeref-state-default>
       <grouping-treeref>true</grouping-treeref>
       <rule>
         <name>include controller</name>
         <module-name>clixon-controller</module-name>
         <operation>enable</operation>
       </rule>
       <rule>
         <name>include myyang</name>
         <module-name>myyang*</module-name>
         <operation>enable</operation>
       </rule>
    </autocli>
  </clixon-config>

What about YANG features?
-------------------------
If you use YANG features that are required to be enabled for your YANG
to function properly, then you need to explicitly enable them in the
configuration file.

Suppose you need the YANG feature: `ietf-ipfix-psamp:exporter`. Edit
the existing `controller.xml` or add a new file
`/usr/local/etc/controller/myfeatures.xml` as follows::

  <?xml version="1.0" encoding="utf-8"?>
  <clixon-config xmlns="http://clicon.org/config">
    <CLICON_FEATURE>ietf-ipfix-psamp:exporter</CLICON_FEATURE>
  </clixon-config>

What about the directory structure?
-----------------------------------

In a typical installation, the main configuration file is in
`/usr/local/etc/clixon/controller.xml`. All other directories are
stated in this configure file.  Extra config files are loaded after
the main in alphabetical order are placed in the
`/usr/local/etc/clixon/controller/` directory. This is useful for
adding and overriding the default config.

The top-level YANG directory is in
`/usr/local/share/clixon/controller`. YANGs are available in the
search path if placed here. Howevere there are three sub-directories
with specific meanings:

  - `main`. Main controller YANGs for the top-level. Note: only place YANGs here if you want them loaded to the top-level.
  - `mounts`. YANGs retrieved from devices are written here
  - `modules`. YANGs for the pyapi

Candidate is locked
-------------------
You may encounter an error when doing commit or connect something like::

  Candidate db is locked by 1677721

This happens if a commit/connect transaction terminates without releasing the candidate datastore.
It should not happen, but if it does, you can unlock candidate with the CLI command::

  cli> transaction unlock

Can Openconfig and IETF YANG co-exist?
--------------------------------------
Yes, but there are some minor notes for the auto-CLI.

YANG openconfig-interfaces and ietf-interfaces have the same top-level `interfaces` symbol but with different
namespaces.

The generated clixon-cli is not fully namespace compliant so one may need to filter the ietf-interfaces if running openconfig.

Filtering the ietf-interfaces is done in the autocli.xml configuratin file as follows::

  <clixon-config xmlns="http://clicon.org/config">
     <autocli>
        <module-default>true</module-default>
        ...
        <rule>
           <name>exclude ietf interfaces</name>
           <module-name>ietf-interfaces</module-name>
           <operation>disable</operation>
        </rule>
     </autocli>
  </clixon-config>
