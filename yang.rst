.. _controller_yang:
.. sectnum::
   :start: 6
   :depth: 3
   
****
YANG
****

Searching
=========

Uniqueness
----------
The controller YANGs rely on uniqueness of revisions. This means that
even with schema mounts, all YANGs are part of the same search domain
wrt revisions. That is, you may not have two different YANGs haveing
the same revision.

Search path
-----------
Because of the uniqueness criterium, the controller uses the same search path for all YANGs. Typically the top-level search path is::

    <CLICON_YANG_DIR>/usr/local/share/clixon</CLICON_YANG_DIR>

This includes all standard YANGs, clixon YANGs and controller YANGs.

Controller YANGs
^^^^^^^^^^^^^^^^
The top-level of the controller-specific YANGs is typically  `/usr/local/share/clixon/controller`.

This can be changed with `configure --with-yang-installdir=DIR`, see Section :ref:`Installation <controller_install>`.

The controller YANG directory has two sub-directories with specific meanings:
  - `main`. Main controller YANGs for the top-level. Note: only place YANGs here if you want them loaded to the top-level. Important: do not place device YANGs there, only controller YANGs.
  - `mounts`. YANGs retreived from devices using RFC 6022 `get-schema` are written here. You can also add local cached YANGs here.

.. note::
        Do not place device YANGs in the `main` directory

Device YANGs
------------
There are two mechanisms to get YANGs from devices:
  1. Dynamic RFC6022 get-schema
  2. Locally defined

Dynamic
^^^^^^^
RFC6022 YANG Module for NETCONF Monitoring defines a protocol for retrieving YANG schemas. Clixon implements this as a main mechanism.

This is automatically invoked if the device advertises "urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring" in the hello protocol. The state-machine mechanism for this is described in Section :ref:`transactions <controller_transactions>`.

Local
^^^^^
You can also declare a module-set which is loaded unconditionally in a device, or device-profile. In the following example, openconfig is declared as locally loaded::

   <devices xmlns="http://clicon.org/controller">
      <device-profile>
         <name>myprofile</name>
         <module-set>
            <module>
               <name>openconfig-interfaces</name>
               <namespace>http://openconfig.net/yang/interfaces</namespace>
            </module>
         </module-set>
      </device-profile>
   </devices>

Note the following:
  1. The locally defined openconfig YANG will searched for using the regular YANG search mechanism (using `CLICON_YANG_DIR`).
  2. If the local YANG is not found, the (connect) transaction will fail.
  3. Any imports declared in a locally defined YANG will also be loaded locally recursively
  4. If the device also supports RFC 6022 get-schema, any further YANGs will be loaded from the device.
  
Structure
=========

Clixon-controller
-----------------
The clixon-controller YANG has the following structure::

   module: clixon-controller
     +--rw processes
     |   +--rw services
     |     +--rw enabled              boolean
     +--rw services
     |   +--rw properties
     +--rw devices
     |   +--rw device-timeout         uint32
     |   +--rw device-group* [name]
     |   | +--rw name                 string
     |   | +--rw description?         string
     |   | +--rw device-group*        leafref
     |   +--rw device-profile* [name]
     |   | +--rw name                 string
     |   | +--rw description?         string
     |   | +--rw user?                string
     |   | +--rw conn-type            connection-type
     |   | +--rw ssh-stricthostkey    boolean
     |   | +--rw yang-config?         yang-config
     |   +--rw device* [name]
     |     +--rw name                 string
     |     +--rw enabled?             boolean
     |     +--rw device-profile       leafref
     |     +--rw description?         string
     |     +--rw user?                string
     |     +--rw conn-type            connection-type
     |     +--rw ssh-stricthostkey    boolean
     |     +--rw yang-config?         yang-config
     |     +--rw device-type          string
     |     +--rw addr                 string
     |     +--ro conn-state           connection-state
     |     +--ro conn-state-timestamp yang:date-and-time
     |     +--ro capabilities
     |     | +--ro capability*        string
     |     +--ro sync-timestamp       yang:date-and-time
     |     +--ro logmsg               string
     |     +--rw config
     +--ro transactions
         +--ro transaction* [tid]
           +--ro tid                  uint64
           +--ro state                transaction-state
           +--ro result               transaction-result
           +--ro description          string
           +--ro origin               string
           +--ro reason               string
           +--ro warning              string
           +--ro timestamp            yang:date-and-time
     notifications:
       +---n services-commit
       |   +--ro tid                  uint64
       +---n controller-transaction
           +--ro tid                  uint64
     rpcs:
         +--config-pull
         +--controller-commit
         +--connection-change
         +--get-device-config
         +--transaction-error
         +--transaction-actions-done
         +--datastore-diff
         +--device-template-apply
  
Service augment
---------------
The services section contains user-defined services not provided by
the controller.  A user adds services definitions using YANG `augment`. For example::

    import clixon-controller { prefix ctrl; }
    augment "/ctrl:services" {
        list myservice {
            ...

Controller-config
-----------------
The clixon-controller-config YANG extends the basic clixon-config with several fields. These have previously been described in Section :ref:`configuration <controller_configuration>`. The structure is as follows::

     module: clixon-controller-config
       augment /cc:clixon-config
       +--rw CONTROLLER_ACTION_COMMAND
       +--rw CONTROLLER_PYAPI_MODULE_PATH
       +--rw CONTROLLER_PYAPI_MODULE_FILTER
       +--rw CONTROLLER_PYAPI_PIDFILE
       +--rw CONTROLLER_YANG_SCHEMA_MOUNT_DIR
