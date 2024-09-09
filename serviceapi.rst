.. _controller_serviceapi:
.. sectnum::
   :start: 10
   :depth: 3
   
***********
Service API
***********

This section describes the `services API`, a protocol using NETCONF+YANG for service code handlers, in particular the `PyAPI`.

See :ref:`Services tutorial <tutorial>` for a step-by-step tutorial on how to create a service.

See the :ref:`Python API <controller_pyapi>` for a detailed description on how Python code is used in PyAPI for the service API.

Overview
========
The service API provides a mechanism to control devices by generating device configurations. The service code uses this API in the following way:

1. Creates a subscription and listens to incoming events (``create-subscription``)
2. Is notified by the controller when a service or service instance configuration has changed (``services-commit``)
3. Edits the device configuration based on service code and possibly other info (``edit-config``)
4. Informs the controller when done (``transactions-actions-done``).

Restrictions
------------
The restrictions on the current service API are as follows:

1. Only a single service code handler is supported, which means that a single process handles all services.
2. One-to-one: One service per object, multiple services may not create the same object

Service model
=============
A service extends the controller YANG as described in the `YANG section <https://clixon-docs.readthedocs.io/en/latest/yang.html>`_ section. For example, a service `ssh-users` may add a new service as follows:

.. code-block:: yang

   module ssh-users {
      namespace "http://clicon.org/ssh-users";
      prefix ssh-users;
      import clixon-controller {
         prefix ctrl;
      }
      revision 2023-05-22 {
         description "Initial prototype";
      }
      augment "/ctrl:services" {
         list ssh-users {   // YANG list
            uses ctrl:created-by-service;
            key instance;
            leaf instance {
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
               leaf role {
	          type string;
	       }
            }
         }
      }
   }

Model details
-------------
Some notes on the `ssh-users` service model, in order:

* A unique service name, ``ssh-users`` which is reflected in the file name and service code.
* A uniqe namespace: ``"http://clicon.org/ssh-users``
* Import the clixon-controller YANG to use constructs as prefixed by ``ctrl:``
* A revision matching the date in the filename.
* The service `augments` the top-level service container in the clixon-controller YANG, i.e., extends it.
* The `list ss-users` with key `instance` defines services instances. A service instance must be declared as a YANG `list` with a single key.
* The instance list must use ``created-by-service`` to keep track of created instances. This is especially important when removing config.

XML configuration
-----------------
An example service encoded as XML for ``ssh-users`` is shown in the following example:

.. code-block:: xml

   <services xmlns="http://clicon.org/controller">
     <ssh-users xmlns="urn:example:test">
        <instance>ops</instance>
        <username>
           <name>eric</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
           <role>admin</role>
        </username>
        <username>
           <name>alice</name>
           <ssh-key>ssh-rsa AAA...</ssh-key>
           <role>guest</role>
        </username>
     </ssh-users>
     <ssh-users xmlns="urn:example:test">
        <instance>devs</instance>
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

This is the format the service normally appears in the controller configuration datastore.

Service instance
-----------------
From the example YANG above, examples of service instances of ``ssh-users`` are::

  ssh-users
  ssh-users[instance='ops']
  ssh-users[instance='devs']

where the first identifies all ``ssh-users`` instances and the other two
identifies the specific instance ``ops`` and ``devs``, respectively.

Device config
=============
The service definition is input to changing the device config, where the actual change is made by
Python code in the PyAPI.

A device configuration could be as follows (inspired by openconfig):

.. code-block:: yang

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

Attributes
==========

The service code typically tags objects it creates by XML attributes. For example:

.. code-block:: xml

   <user cl:creator="ssh-users[instance='testuser']" nc:operation="merge" xmlns:cl="http://clicon.org/lib">
      <username>testuser</username>
	 <config>
	    <username>testuser</username>
	    <ssh-key>AAAAB3NzaC...</ssh-key>
	    <role>admin</role>
	 </config>
      </username>
   </user>

Note that the two attributes:

* ``nc:operation="merge"`` : the NETCONF edit operation
* ``cl:creator="ssh-users[instance='testuser']"`` : the creator tag.

Operator
--------
The service code sends NETCONF ``edit-config`` RPCs to the controller to create and modify the device configuration tree. Edit-config operations are typically ``merge`` which is the default NETCONF operation.

Other NETCONF operations are described here: `RFC 6241 <https://www.rfc-editor.org/rfc/rfc6241.html#section-7.2>`_, most of which are not applicable.

Creator
-------
The ``creator`` tag is an XPath used to keep track of which service instance
have created which configuration object. This is further described in section `Creator tags`_.

Creator tags
============
The stateless operation of the service code requires that the controller understands which XML objects are created, and by which service instance.

It works in the following way:

* The user edits some service instances (add/edit/remove), using the CLI anc commits
* The controller then removes all configuration objects tagged with the services instances
* The service code is triggered and (re)generates all device configuration of the service instances
* The controller computes the difference of the generated config with the existing device config.
* The controller pushes the modifications to the devices

Example
-------
In the following example using the XML in Section `XML configuration`_, three device objects (usernames eric, alice and kim) are tagged with service instances in one device ``A``, as follows:

.. table:: `Device A with service-instance tags`
   :widths: auto
   :align: left

   =============  =======================
   Device object  Service-instance
   =============  =======================
   eric           ssh-users[instance='ops']
   alice          ssh-users[instance='devs']
   kim            ssh-users[instance='ops'],
   =============  =======================

where device objects `eric` and `kim` are created by service instance `ops` (more precisely `ssh-users[instance='ops']`) and `alice` is created by `devs`.

Suppose that service instance `ops` is deleted, then all device objects tagged with `ops` are deleted:

.. table:: `Device A after removal of ops`
   :widths: auto
   :align: left
            
   =============  =======================
   Device object  Service-instance
   =============  =======================
   alice          ssh-users[instance='devs']
   =============  =======================

Note also that this example only considers a single device `A`. In reality there are many more devices.

Algorithm
=========
The algorithm for managing device objects using creator tags is as follows. Consider a commit operation where some services have changed by adding, deleting or modifying service -instances:

  1. The controller makes a diff of the candidate and running datastore and identifies all changed services-instances
  2. For all changed service-instances S:
    
    - For all device nodes D tagged with that service-instance tag:

      - If S is the only tag, delete D
      - Otherwise, delete the tag, but keep D

  3. The controller sends a notification to the PYAPI including a list of modified service-instances S
  4. The PyAPI creates device objects based on the service instances S, merges with the datastore and commits
  5. The controller makes a diff between the modified datastore and running and pushes to the devices

The algorithm is `stateless` in the sense that the PyAPI recreates all
objects of the modified service-instances. If a device object is not
created, it is considered as deleted by the controller. Keeping track
of deleted or changed service-instances is done only by the
controller.
     
Protocol
========
The following diagram shows an overview of the service API protocol::

     Backend       Service API            Service code (eg PyAPI)
        |                                      |
        + <--- <create-subscription> ---       +
        |                                      |
        +  --- <services-commit> --->          +
        |                                      |
        + <---   <edit-config>   ---           +
        |            ...                       |
        + <---   <edit-config>   ---           +
        |                                      |
        + <--- <transactions-actions-done> --- +
        |                                      |
        |          (wait)                      |
        +  --- <services-commit> --->          +
        |            ...                       |
           
where each message is described by the following text.
        
Registration
------------
The service code registers subscriptions of service commits by using RFC 5277
notification streams:

.. code-block:: xml

    <create-subscription>
       <stream>service-commit</stream>
    </create-subscription>

Notification
------------
Thereafter, controller notifications of type `service-commit` are sent
from the backend to the service code every time a
`controller-commit` RPC is initiated with an `action` component. This
is typically done when CLI commands `commit push`, `commit diff` and
others are made.

An example of a `service-commit` notification is the following:

.. code-block:: xml

    <services-commit>
       <tid>42</tid>
       <source>candidate</source>
       <target>actions</target>
       <service>ssh-users[instance='ops']</service>
       <service>ssh-users[instance='devs']</service>
    </services-commit>

In the example above, the transaction-id is `42` and the services definitions are read from
the `candidate` datastore. Updated device edits are written to the `actions` datastore.

The notification also informs the service code that two service instances have changed.

A special case is if `no` service-instance entries are present. If so, it means
`all` services in the configuration should be re-applied.


Editing
-------
In the following example, the PyAPI adds an object in the device configuration tagged with the service instance `ssh-users[instance='ops']`:

.. code-block:: xml

  <edit-config>
    <target><actions xmlns="http://clicon.org/controller"/></target>
    <config>
      <devices xmlns="http://clicon.org/controller">
        <device>
          <name>A</name>
          <config>
            <users xmlns="urn:example:users" xmlns:cl="http://clicon.org/lib" nc:operation="merge">
              <user cl:creator="ssh-users[instance='ops']">
                <username>alice</username>>
                <ssh-key>ssh-rsa AAA...</ssh-key>
              </user>
          </users>
          </config>
        </device>
      </devices>
    </config>
  </edit-config>

Note that the servic code needs to make a `get-config` to read the
service definition.  Further, there is no information about what
changes to the services have been made. The idea is that the service code
reapplies a changed service and the backend sorts out any
deletions using the tagging mechanism.

Finishing
---------
When all modifications are done, the service code issues a `transaction-actions-done` message to the backend:

.. code-block:: xml

    <transaction-actions-done xmlns="http://clicon.org/controller">
      <tid>42</tid>
    </transaction-actions-done>

After the `done` message has been sent, no further edits are made by
the service code, it waits for the next notification.

The backend, in turn, pushes the edits to the devices, or just shows
the diff, or validates, depending on the original request parameters.

Error
-----
The service code can also issue an error to abort the transaction. For example:
  
.. code-block:: xml

   <transaction-error>
      <tid>42</tid>
      <origin>pyapi</origin>
      <reason>No connection to external server</reason>
   </transaction-error>

In this case, the backend terminates the transaction and signals an error to the originator, such as a CLI user.
    
Another source of error is if the backend does not receive a `done`
message. In this case it will eventually timeout and also signal an error.
