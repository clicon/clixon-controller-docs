.. _tutorial:
.. sectnum::
   :start: 6
   :depth: 3

****************
Service tutorial
****************

This tutorial guides you through the process of creating a simple
service using the OpenConfig demo environment. The service is built using two parts:

1. YANG specification describing the service.
2. Python code implementing the service logic.

The service creates users and distributes SSH keys to the devices.

Prerequisites
=============
Before you start, you need to have setup as described in the :

1. A `host` where the controller runs
2. Two `devices` accessible by NETCONF over SSH.

How to setup this environment is described in :ref:`setup tutorial <setup_tutorial>`.

Service model
=============

Each service is described using a `YANG` model. YANG is a data modeling
language used to model configuration data, state data, remote
procedure calls and notifications for network management protocols. If
you want to know more about YANG, a detailed description is `RFC 7950
<https://www.rfc-editor.org/rfc/rfc7950.html>`_.

The YANG model for the
service is in directory `/usr/local/share/clixon/controller/main/` and
is named with the service name. In this example the service is named
`ssh-users` and the YANG model is in
`/usr/local/share/clixon/controller/main/ssh-users@2023-05-22.yang`. If
the YANG file is modified, the controller must be restarted to load
the new YANG file.



The YANG model used for this example is a simple model that allows the
tconfiguration of SSH users on the devices. The model is defined in the
file `ssh-users.yang`:

.. code-block:: yang

   module ssh-users {
       namespace "http://clicon.org/ssh-users";
       prefix ssh-users;

       import clixon-controller { prefix ctrl; }

       revision 2023-05-22 {
	   description "Initial prototype";
       }

       augment "/ctrl:services" {
	   list ssh-users {
	       uses ctrl:created-by-service;

	       key instance;
	       leaf instance {
		   type string;
	       }

	       description "SSH users service";

	       list username {
		   key name;
		   leaf name {
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

The YANG model above describes a service that can be used to configure
SSH users on the devices. The service is named `ssh-users` and has a
list of users. Each user has a name, an SSH key and a role.

Service CLI commands
====================

When the YANG file is added new CLI commands are available in
the CLI. The CLI commands are generated from the YANG file. The CLI
commands are used to configure the service. The CLI commands are:

.. code-block:: bash

   $ clixon_cli
   user@test> configure
   user@test[/]# set services ?
   user@test[/]# set services
     <cr>
     properties
     ssh-users             SSH users service
   user@test[/]# set services ssh-users ?
     <instance>
   user@test[/]# set services ssh-users test ?
     <cr>
     created               List of created objects used by services.
     username

To configure a new ssh-user the full sequence of CLI commands are:

.. code-block:: bash

   user@test[/]# set services ssh-users test
   user@test[/]# set services ssh-users test username testuser ssh-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQ6..."
   user@test[/]# set services ssh-users test username testuser role admin

When the service is configured in the CLI the command `commit diff`
executes the Python code which we will write in the next step. The
Python code will configure the devices with the new user and when the
output looks good the command `commit` is executed to save the
configuration and push it to the devices.

Python
------

The Python code is in the directory
`/usr/local/share/clixon/controller/modules/` and is named with the
service name. In this example the service is named `ssh-users` and the
Python code is in
`/usr/local/share/clixon/controller/modules/ssh_users.py`. If the
Python file is modified, the controller or the API server must be
restarted to load the new Python file.

The goal of this step is to write Python code which generates the
following NETCONF XML on the devices:

.. code-block:: xml

   <system xmlns="http://openconfig.net/yang/system">
      <aaa>
	 <authentication>
	    <users>
	       <user>
		  <username>new_username</username>
		  <config>
		     <username>new_username</username>
		     <ssh-key>ssh key AAAAA</ssh-key>
		     <role>operator</role>
		  </config>
	       </user>
	    </users>
	 </authentication>
      </aaa>
   </system>

Code breakdown
==============

Each service has a Python file which contains the Python code for the
service. When the code is executed the API server will start with the
function `setup` and the arguments `root`, `log` and `**kwargs`. The
`root` argument is the root of the configuration data tree. The `log`
argument is a logger object which can be used to log messages. The
`**kwargs` argument is a dictionary with additional arguments such as
the name of the service instance.

First we need to import the necessary modules:

.. code-block:: python

   from clixon.element import Element
   from clixon.parser import parse_template
   from clixon.helpers import get_service_instance

The Element module is used to create new XML elements in the
configuration data tree. The parse_template module is used to parse
the XML template. The get_service_instance module is used to get the
service instance.

Each service module _must_ have a variable named `SERVICE` which is
the name of the service. The name should correspond to the name of the
YANG model associated with the service.

.. code-block:: python

   SERVICE = "ssh-users"

The first function in the Python code is the `setup` function. The
first thing we do in the setup function is to check whether the
service is configured. If the service is not configured we return and
do nothing.

.. code-block:: python

   def setup(root, log, **kwargs):
      # Check if the service is configured
      try:
	 _ = root.services.ssh_users
      except AttributeError:
	 return

Next step is to get the service instance. To do this we can use the
helper function `get_service_instance` which will return an
configuration data tree element with the service instance if it exists
other wise it will return None.

.. code-block:: python

      # Get the service instance
      service_instance = get_service_instance(root, SERVICE, **kwargs)

      # Check if the instance is the one we are looking for
      if service_instance is None:
	 return

Next step is to get the username, ssh-key and role from the service
instance. To do this we iterate over the service instance and get the
values.

.. code-block:: python

   # Get the data from the user
   service_name = instance.service_name.get_data()
   username = user.name.get_data()
   ssh_key = user.ssh_key.get_data()
   role = user.role.get_data()

The next step is to create the XML template for the new user. The XML
template is a string with placeholders for the username, ssh-key and
role. The placeholders are replaced with the values from the service
instance when the template is parsed.

.. code-block:: python

   # Create the XML for the new user
   new_user = parse_template(USER_XML,
			     SERVICE_NAME=service_name,
			     USERNAME=username,
			     SSH_KEY=ssh_key,
			     ROLE=role).user

We then check if the needed elements in the configuration data tree
are present. If they are not present we create them.

.. code-block:: python

   # Add the new user to all devices
   for device in root.devices.device:
      # Check if the device has the system element
      if not device.config.system.get_elements("aaa"):
	 device.config.system.create("aaa")

      # Check if the device has the authentication element
      if not device.config.system.aaa.get_elements("authentication"):
	 device.config.system.aaa.create("authentication")

      # Check if the device has the users element
      if not device.config.system.aaa.authentication.get_elements("users"):
	 device.config.system.aaa.authentication.create("users")

And finally we add the new user to the configuration data tree.

.. code-block:: python

   # Add the new user to the device
   device.config.system.aaa.authentication.users.add_element(new_user)

Full service Python code
========================

The full Python code for this example service looks like this:

.. code-block:: python

   from clixon.element import Element
   from clixon.parser import parse_template
   from clixon.helpers import get_service_instance

   SERVICE = "ssh-users"

   # The XML template for the new user
   USER_XML = """
   <user cl:creator="ssh-users[service-name='{{SERVICE_NAME}}']" nc:operation="merge" xmlns:cl="http://clicon.org/lib">
      <username>{{USERNAME}}</username>
	 <config>
	    <username>{{USERNAME}}</username>
	    <ssh-key>{{SSH_KEY}}</ssh-key>
	    <role>{{ROLE}}</role>
	 </config>
   </user>
   """

   def setup(root, log, **kwargs):
      # Check if the service is configured
      try:
	 _ = root.services.ssh_users
      except Exception:
	 return

      # Get the service instance
      instance = get_service_instance(root,
				      service_name,
				      instance=kwargs["instance"])

      # Check if the instance is the one we are looking for
      if not instance:
	 return

      # Iterate all users in the instance
      for user in instance.username:

	 # Get the data from the user
	 service_name = instance.service_name.get_data()
	 username = user.name.get_data()
	 ssh_key = user.ssh_key.get_data()
	 role = user.role.get_data()

	 # Create the XML for the new user
	 new_user = parse_template(USER_XML,
				   SERVICE_NAME=service_name,
				   USERNAME=username,
				   SSH_KEY=ssh_key,
				   ROLE=role).user

	 # Add the new user to all devices
	 for device in root.devices.device:
	    # Check if the device has the system element
	    if not device.config.system.get_elements("aaa"):
	       device.config.system.create("aaa")

	    # Check if the device has the authentication element
	    if not device.config.system.aaa.get_elements("authentication"):
	       device.config.system.aaa.create("authentication")

	    # Check if the device has the users element
	    if not device.config.system.aaa.authentication.get_elements("users"):
	       device.config.system.aaa.authentication.create("users")

	    # Add the new user to the device
	    device.config.system.aaa.authentication.users.add(new_user)

When the Python code above is written to the file
`/usr/local/share/clixon/controller/modules/ssh_users.py` the service
API server must be restarted to load the new Python file. This can be
done either by restarting the controller or by restarting the service
API server:

.. code-block:: bash

   $ clixon_cli
   user@test> ser
   user@test> processes services restart
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <ok xmlns="http://clicon.org/lib"/>
   </rpc-reply>

And then we can configure the service in the CLI and commit the
configuration. When the configuration is committed the Python code is
executed and the new user is added to the devices:

.. code-block:: bash

   $ clixon_cli
   user@test> configure
   user@test[/]# set services ssh-users test
   user@test[/]# set services ssh-users test username testuser ssh-key "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQ6..."
   user@test[/]# set services ssh-users test username testuser role admin
   user@test[/]# commit diff
   openconfig1:
	       <users xmlns="http://openconfig.net/yang/system">
   +              <user>
   +                 <username>testuser</username>
   +                 <config>
   +                    <username>testuser</username>
   +                    <ssh-key>ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQ6...</ssh-key>
   +                    <role>admin</role>
   +                 </config>
   +              </user>
	       </users>
   OK

To save the configuration and push it to the devices the command
`commit` is executed. Then the Python code is executed again and the
new user is pushed to the devices:

.. code-block:: bash

   user@test[/]# commit
   OK

The user can also be removed from the devices by deleting the service
and committing the configuration.

.. code-block:: bash

   user@test[/]# delete services ssh-users test
   user@test[/]# commit diff
   openconfig1:
	       <users xmlns="http://openconfig.net/yang/system">
   -              <user>
   -                 <username>testuser</username>
   -                 <config>
   -                    <username>testuser</username>
   -                    <ssh-key></ssh-key>
   -                    <role>admin</role>
   -                 </config>
   -              </user>
	       </users>
   OK
   user@test[/]# commit

The Python code above is a simple example of how to configure a new
user on the devices. The Python code can be extended to handle more
complex configurations and to handle more services. The Python code
can also be extended to handle more devices and to handle more
configuration elements on the devices.
