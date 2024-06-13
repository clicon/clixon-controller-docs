.. _controller_serviceapi:
.. sectnum::
   :start: 8
   :depth: 3
   
***********
Service API
***********

The controller provides an `service API` which is a YANG-defined protocol for external action handlers, including the `PyAPI`.

The backend implements a tagging mechanism to keep track of what parts
of the configuration tree were created by which services.  In this
way, reference counts are maintained so that objects can be removed in
a correct way if multiple services create the same object.

There are some restrictions on the current service API:

1. Only a single action handler is supported, which means that a single action handler handles all services.
2. The algorithm is not hierarchical, that is, if there is a tag on a device object, tags on children are not considered
3. One-to-one: One service per object, multiple services may not create the same object

Service instance
----------------
A service extends the controller yang as described in the `YANG section <https://clixon-docs.readthedocs.io/en/latest/yang.html>`_ section. For example, a service `ssh-users` may augment the original as follows::

   augment "/ctrl:services" {
      list ssh-users {   // YANG list
         key group;      // Single key
         leaf group {
            type string;
	 }
         list username {
            key name;
            leaf name{
               type string;
            }
            leaf ssh-key {
               type string;
            }
         }
      }
   }

The service must be on the following form:

1. The top-level is a YANG list (eg `ssh-users` above)
2. The list has a single key (eg `group` above)

The rest of the augmented service can have any form (eg `list username` above).
   
.. note::
        An augmented service must start with a YANG list with a single key

An example service XML for `ssh-users` is::

   <services xmlns="http://clicon.org/controller">
     <ssh-users xmlns="urn:example:test">
        <group>ops</group>
        <username>
           <name>eric</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
        <username>
           <name>alice</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
     </ssh-users>
     <ssh-users xmlns="urn:example:test">
        <group>devs</group>
        <username>
           <name>kim</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
        <username>
           <name>alice</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
        </username>
     </ssh-users>
   </services>

The service protocol defines a service instances as::

  <list>  |  <list>[<key>='<value>']

From the example YANG above, examples of service instances of `ssh-users` are::

  ssh-users
  ssh-users[group='ops']
  ssh-users[group='devs']

where the first identifies all `ssh-users` instances and the other two
identifies the specific instances given above

Device config
-------------
The service definition is input to changing the device config, where the actual change is made by
Python code in the PyAPI.

A device configuration could be as follows (inspired by openconfig)::

  container users {
     description "Enclosing container list of local users";
     list user {
        key "username";
        description "List of local users on the system";
        leaf username {
            type string;
            description "Assigned username for this user";
        }
        leaf ssh-key {
            type string;
            description "SSH public key for the user (RSA or DSA)";
        }
     }
  }

Tags
----
An action handler tags device configuration objects it creates with the name of the service instances
using the `cl:creator` YANG extension.  This is used to track which instance created
an object. Only one service per created object is supported.

In the following example, three device objects are tagged with service instances in one device, as follows:

.. table:: `Device A with service-instance tags`
   :widths: auto
   :align: left

   =============  =======================
   Device object  Service-instance
   =============  =======================
   eric           ssh-users[group='ops']
   alice          ssh-users[group='devs']
   kim            ssh-users[group='ops'],
                  ssh-users[group='devs']
   =============  =======================

where device objects `eric` and `alice` are created by service instance `ops` (more precisely `ssh-users[group='ops']`) and `devs` respectively, and `kim` is created by both.

Suppose that service instance `ops` is deleted, then all device objects tagged with `ops` are deleted:

.. table:: `Device A after removal of ops`
   :widths: auto
   :align: left
            
   =============  =======================
   Device object  Service-instance
   =============  =======================
   alice          ssh-users[group='devs']
   kim            ssh-users[group='devs']
   =============  =======================

Note that `kim` still remains since it was created by both ops and devs.

Note also that this example only considers a single device `A`. In reality there are many more devices.

Example python
--------------
An example PyAPI script takes the service ssh-users definition and creates users on the actual devices, for example::

    for instance in root.services.users:
        for user in instance.username:
            username = ssh-users.name.cdata
            ssh_key = ssh-users.ssh_key.cdata
            for device in root.devices.device:
                new_user = Element("user",
                                   attributes={
                                       "cl:creator": "users[group='ops']",
                                       "nc:operation": "merge",
                                       "xmlns:cl": "http://clicon.org/lib"})
                new_user.create("name", cdata=username)
                new_user.create("authentication")
                new_user.authentication.create("ssh-rsa")
                new_user.authentication.ssh_rsa.create("name", cdata=ssh_key)
                device.config.configuration.system.login.add(new_user)


Algorithm
---------
The algorithm for managing device objects using tags is as follows. Consider a commit operation where some services have changed by adding, deleting or modifying service -instances:

  1. The controller makes a diff of the candidate and running datastore and identifies all changed services-instances
  2. For all changed service-instances S:
    
    - For all device nodes D tagged with that service-instance tag:

      - If S is the only tag, delete D
      - Otherwise, delete the tag, but keep D

  3. The controller sends a notification to the PYAPI including a list of modified service-instances S
  4. The PyAPI creates device objects based on the service instances S, merges with the datastore and commits
  5. The controller makes a diff between the modified datastore and running and pushes to the devices

The algorithm is stateless in the sense that the PyAPI recreates all
objects of the modified service-instances. If a device object is not
created, it is considered as deleted by the controller. Keeping track
of deleted or changed service-instances is done only by the
controller.
     
Protocol
--------
The following diagram shows an overview of the action protocol::

     Backend                           Action handler
        |                                  |
        + <--- <create-subscription> ---   +
        |                                  |
        +  --- <services-commit> --->      +
        |                                  |
        + <---   <edit-config>   ---       +
        |            ...                   |
        + <---   <edit-config>   ---       +
        |                                  |
        + <---  <trans-actions-done> ---   +
        |                                  |
        |          (wait)                  |
        +  --- <services-commit> --->      +
        |            ...                   |           
           
where each message will be described in the following text.
        
Registration
^^^^^^^^^^^^
An action handler registers subscriptions of service commits by using RFC 5277
notification streams::

    <create-subscription>
       <stream>service-commit</stream>
    </create-subscription>

Notification
^^^^^^^^^^^^
Thereafter, controller notifications of type `service-commit` are sent
from the backend to the action handler every time a
`controller-commit` RPC is initiated with an `action` component. This
is typically done when CLI commands `commit push`, `commit diff` and
others are made.

An example of a `service-commit` notification is the following::

    <services-commit>
       <tid>42</tid>
       <source>candidate</source>
       <target>actions</target>
       <service>ssh-users[group='ops']</service>
       <service>ssh-users[group='devs']</service>
    </services-commit>

In the example above, the transaction-id is `42` and the services definitions are read from
the `candidate` datastore. Updated device edits are written to the `actions` datastore.

The notification also informs the action server that two service instances have changed.

A special case is if `no` service-instance entries are present. If so, it means
`all` services in the configuration should be re-applied.


Editing
^^^^^^^
In the following example, the PyAPI adds an object in the device configuration tagged with the service instance `ssh-users[group='ops']`::

  <edit-config>
    <target><actions xmlns="http://clicon.org/controller"/></target>
    <config>
      <devices xmlns="http://clicon.org/controller">
        <device>
          <name>A</name>
          <config>
            <users xmlns="urn:example:users" xmlns:cl="http://clicon.org/lib" nc:operation="merge">
              <user cl:creator="ssh-users[group='ops']">
                <username>alice</username>>
                <ssh-key>ssh-rsa AAA...</ssh-key>
              </user>
          </users>
          </config>
        </device>
      </devices>
    </config>
  </edit-config>

Note that the action handler needs to make a `get-config` to read the
service definition.  Further, there is no information about what
changes to the services have been made. The idea is that the action
handler reapplies a changed service and the backend sorts out any
deletions using the tagging mechanism.

Finishing
^^^^^^^^^
When all modifications are done, the action handler issues a `transaction-actions-done` message to the backend::

    <transaction-actions-done xmlns="http://clicon.org/controller">
      <tid>42</tid>
    </transaction-actions-done>

After the `done` message has been sent, no further edits are made by
the action handler, it waits for the next notification.

The backend, in turn, pushes the edits to the devices, or just shows
the diff, or validates, depending on the original request parameters.

Error
^^^^^
The action handler can also issue an error to abort the transaction. For example::
  
    <transaction-error>
      <tid>42</tid>
      <origin>pyapi</origin>
      <reason>No connection to external server</reason>
    </transaction-error>

In this case, the backend terminates the transaction and signals an error to the originator, such as a CLI user.
    
Another source of error is if the backend does not receive a `done`
message. In this case it will eventually timeout and also signal an error.
