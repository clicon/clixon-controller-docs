.. _tutorial:
.. sectnum::
   :start: 11
   :depth: 3

****************
Service tutorial
****************

This tutorial will guide you through the process of creating a simple
service using our OpenConfig demo environment. The service is built using two parts:

1. YANG specification describing the service.
2. Python code implementing the service logic.

The service creates users and distributes SSH keys to the devices.

Preequisites
------------


1. Either a virtual machine or a physical machine where you have root
   access.
2. Docker installed on the machine.

Environment
----------

The environment used consists of one controller and two OpenConfig
devices. The controller communicates with the devices using NETCONF
tunneled over SSH and the user interacts with the controller using
a CLI.

The controller is running on the host machine and the devices are
running in Docker containers. See the :ref:`Installation
<controller_install>` section for more information on how to set up
the controller.

Start the device:

.. code-block:: bash

   $ docker run --name openconfig1 -it clixon/openconfig

Add your local root users key to the authorized_keys file in the device:

.. code-block:: bash

   $ docker exec -it openconfig1 sh
   # su - noc
   # echo "<your public key>" >> ~/.ssh/authorized_keys
   # chown noc:noc ~/.ssh/authorized_keys
   # chmod 400 ~/.ssh/authorized_keys

Then verify that you can log in to the device using your key and the
NETCONF subsystem is running:

Get the IP address of the device:

.. code-block:: bash

   $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' openconfig1

The address is for example 172.17.0.2. And then log in to the device:
   
.. code-block:: bash

   $ sudo ssh noc@<IP address from above> -s netconf
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability><capability>urn:ietf:params:netconf:base:1.0</capability><capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=0</capability><capability>urn:ietf:params:netconf:capability:candidate:1.0</capability><capability>urn:ietf:params:netconf:capability:validate:1.1</capability><capability>urn:ietf:params:netconf:capability:startup:1.0</capability><capability>urn:ietf:params:netconf:capability:xpath:1.0</capability><capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all,trim,report-all-tagged</capability><capability>urn:ietf:params:netconf:capability:notification:1.0</capability><capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring</capability></capabilities><session-id>2</session-id></hello>]]>]]>

Repeat the steps for the second device and name it openconfig2.

Add the devices to the controller:

.. code-block:: bash

   $ clixon_cli
   user@test> configure
   user@test[/]# set device device openconfig1 addr 172.17.0.2
   user@test[/]# set device device openconfig1 user noc
   user@test[/]# set device device openconfig1 conn-type NETCONF_SSH
   user@test[/]# set device device openconfig2 addr 172.17.0.3
   user@test[/]# set device device openconfig2 user noc
   user@test[/]# set device device openconfig2 conn-type NETCONF_SSH
   user@test[/]# commit local
   user@test[/]# exit

And then connect to the devices:

.. code-block:: bash

   user@test> connection open
   user@test> show connections
   Name                    State      Time                   Logmsg
   ================================================================
   openconfig1             OPEN       2024-09-02T14:15:59
   openconfig2             OPEN       2024-09-02T14:15:59

   
YANG
----

Each service is described using a YANG model. The YANG model for the
service is in directory `/usr/local/share/clixon/controller/main/` and
is named with the service name. In this example the service is named
`ssh-users` and the YANG model is in
`/usr/local/share/clixon/controller/main/ssh-users@2023-05-22.yang`. If
the YANG file is modified, the controller must be restarted to load
the new YANG file.

If you want to know more about YANG, see the RFC 7950.

The YANG for this example service looks like this:

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

When the YANG file is added new CLI commands are available in the CLI:

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


The Python code for this example service looks like this:

.. code-block:: python

   from clixon.element import Element
   from clixon.parser import parse_template
  
   SERVICE = "ssh-users"
  
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
         _ = root.services
      except Exception:
         return

      # Iterate all service instances
      for instance in root.services.ssh_users:
         # Check if the instance is the one we are looking for
         if instance.service_name != kwargs["instance"]:
            continue

	 # Iterate all users in the instance
         for user in instance.username:
            service_name = instance.service_name.get_data()
            username = user.name.get_data()
            ssh_key = user.ssh_key.get_data()
            role = user.role.get_data()

	    # Create the XML for the new user
            new_user = parse_template(USER_XML, SERVICE_NAME=service_name,
                                      USERNAME=username, SSH_KEY=ssh_key, ROLE=role).user

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
`/usr/local/share/clixon/controller/modules/ssh_users.py` the service API server must be restarted to load the new Python file. This can be done either by restarting the controller or by restarting the service API server:

.. code-block:: bash

   $ clixon_cli
   user@test> ser
   user@test> processes services restart
   <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
      <ok xmlns="http://clicon.org/lib"/>
   </rpc-reply>

And then we can configure the service in the CLI and commit the configuration:

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

