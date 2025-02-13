.. _controller_extensions:
.. sectnum::
   :start: 14
   :depth: 3

**********
Extensions
**********

Introduction
============

Extensions are a way to alter existing YANG models or extend the controller's (binary) callbacks.

There are two kinds of extensions:
- YANG extensions: to add or modify device YANG
- Plugin extensions: to add device-specific code

YANG extensions are useful if the YANG provided by a device needs to
add new features to a model or to modify existing models.

YANG extensions
===============

Creating a YANG extension
-------------------------
To create an extension, you need to create a new file under (for example)
``/usr/local/share/controller/common/extensions/``. The file name
should contain a YANG revision and end with ``.yang``. As an example,
let's create an extension that limits the length of the ``hostname``
leaf in the ``ietf-system`` model to 10 characters.

Using the OpenConfig example device provided by the controller we can
configure a hostname longer than 10 characters like this::

  > clixon_cli
  cli> configure
  cli[/]# set devices device openconfig1 config system config hostname test
  cli[/]# commit diff
  openconfig1:
      <config xmlns="http://openconfig.net/yang/system">
  -       <hostname>openconfig1</hostname>
  +       <hostname>abcdefghijkl</hostname>
      </config>
  OK
  cli[/]# commit
  OK

The full configuration of the device can be seen with the following command::

  cli> show configuration devices device r1 config system config hostname
  <!-- openconfig1: -->
  <devices xmlns="http://clicon.org/controller">
     <device>
	<name>openconfig1</name>
	<config>
	   <system xmlns="http://openconfig.net/yang/system">
	      <config>
		 <hostname>openconfig1</hostname>
	      </config>
	   </system>
	</config>
     </device>
  </devices>


To create a new YANG extension we first have to create a new file in the 
``common/`` directory, like this: ``/usr/local/share/controller/common/extensions/test@2025-02-30.yang`` and add the following content::

  module test {
      namespace "http://clicon.org/controller/extensions/test";
      prefix test;
  
      import openconfig-system {
          prefix sys;
      }
  
      import openconfig-system {
          prefix oc-sys;
      }
  
      revision 2025-02-28 {
          description "Limit the length of the hostname to 10 characters";
      }
  
      deviation "/oc-sys:system/oc-sys:config/oc-sys:hostname" {
          deviate replace {
              type uint32 {
              }
          }
      }
  }

The YANG specification above will augment the configuration which is
referenced by the path. In this case, the path is
``/sys:system/sys:hostname``. The deviation replaces the type of the
hostname leaf with a uint32. This will result in an error if the
hostname already has a value that is not a number.

The tricky part when creating an extension is to know the path to the
leaf you want to augment. The path is the same as the path in the YANG
model, but with the prefix of the module that defines the leaf. In
this case, the path is ``/sys:system/sys:hostname`` because the leaf
is defined in the ``ietf-system`` model and the prefix is ``sys``.


The last thing is to add the following configuration in the controller CLI::

  cli[/]# set devices device openconfig1 module-set module test namespace http://clicon.org/controller/extensions/test
  cli[/]# commit local


The configuration above tells the controller to apply the extension
``test@2025-02-30`` to the device ``openconfig1``. The same
configuration can also be applied to device-profiles if you want to
apply the extension to all devices that use the device-profile.

Ignoring configuration
----------------------
In some cases, you might want to ignore the configuration of a
leaf. For example if the device adds configuration or hashes
configuration. To ignore the configuration of a leaf you can use the
``cl:ignore-compare`` statement.

The following example shows how to ignore the ``uid`` leaf added by
JunOS when a new user is created::

  module controller-extensions-uid {
      namespace "http://clicon.org/ext/uid";
  
      prefix cl-ext;
  
      import clixon-lib {
          prefix cl;
      }
  
      import junos-conf-root {
  	prefix jc;
      }
  
      import junos-conf-system {
  	prefix jcs;
      }
  
      revision 2024-01-01 {
  	description "Initial prototype";
      }
  
      augment "/jc:configuration/jcs:system/jcs:login/jcs:user/jcs:uid" {
  	cl:ignore-compare;
      }
  }

Plugin extensions
=================
A plugin extension builds a dynamic loadable module (``.so``) which adds plugin code to the controller. This is an advanced feature.

Adding a new plugin is done by adding .c code under the ``plugins/`` dir and then doing::

  > cd plugin
  > make
  > sudo make install

You may need to edit the Makefile.
  
For example, if you add the plugin: ``myplugin.be.c``, a ``myplugin.be.so`` will be installed in the libdir along with ``controller.so``.

The new ``myplugin.be.so`` plugin will then be loaded alongside the controller plugin. Note that loading is made alphabeticaly, in case you want to insert your plugin before or after the main plugin.

Adding a plugin can be useful if you need to add code to handle some device-specific behaviour, such as::

  - Adding an extra validation or commit action
  - Intercept RPC:s with wrap code
  - Translate XML

The plugins follow the regular Clixon plugins. The controller itself is a plugin, and adding an extension plugin is similar to modifying the controller code. However, it adds code in a modular fashion which is easier to maintain than changing the source code.

