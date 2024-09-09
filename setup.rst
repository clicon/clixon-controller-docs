.. _setup_tutorial:
.. sectnum::
   :start: 4
   :depth: 3

**************
Setup tutorial
**************

The aim of this tutorial is to setup the following:

1. A controller host,
2. Two openconfig devices using docker
3. NETCONF connections from the controller to both devices using SSH

You need such a setup if you want to adapt the controller to your own devices or develop a service.

You can skip this tutorial if you only want to run the controller CLI using the docker demo.

Prerequisites
=============
Before you start, you need to have the following:

1. Either a virtual machine or a physical machine where you have root
   access (the host). Typically a Linux OS.
2. Docker installed on the machine.

Controller installation
=======================
Setup the host using the instructions in the :ref:`Installation <controller_install>` section. The following should be installed:

* CLIgen
* Clixon
* Python API
* Clixon controller

SSH key generation
==================
Generate a root SSH key pair if you do not already have one.  The public key will be added to the
devices. To see if you have an SSH key pair, check the contents of the
``.ssh`` directory in your root directory:

.. code-block:: bash

   $ sudo ls /root/.ssh
   authorized_keys  id_rsa  id_rsa.pub  known_hosts

If you do not have an SSH key pair, generate one using the following
command:

.. code-block:: bash

   $ sudo ssh-keygen -C "controller key" -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""

To get your public key which we will use later, run the following
command:

.. code-block:: bash

   $ sudo cat /root/.ssh/id_rsa.pub

.. note::
   The controller uses root keys. Work is ongoing to use non-root keys.


Docker setup
============
The devices run as Docker containers. The Docker image is `clixon/openconfig` is a
pre-built image containing the OpenConfig models. The `noc` user is used to access the container.

The image is available on
Docker Hub and can be pulled using the following command:

.. code-block:: bash

   $ docker pull clixon/openconfig

To start the devices, run the following commands:

.. code-block:: bash

   $ docker run -d --name openconfig1 -it clixon/openconfig
   $ docker run -d --name openconfig2 -it clixon/openconfig

The devices are visible in the list of running containers:

.. code-block:: bash

   $ docker ps
   CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                      NAMES
   1b3b3b3b3b3b        clixon/openconfig   "/bin/sh"           2 minutes ago       Up 2 minutes       22/tcp, 80/tcp, 830/tcp    openconfig1
   2a2a2a2a2a2a        clixon/openconfig   "/bin/sh"           2 minutes ago       Up 2 minutes       22/tcp, 80/tcp, 830/tcp    openconfig2

The devices are running. The next step is to get their address, install keys and then connect to them.

IP address
----------
Find the IP addresses of the devices:

.. code-block:: bash

   $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' openconfig1
   $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' openconfig2

The addresses are for example ``172.17.0.2`` and ``172.17.0.3``.

Add keys
--------
Add the root public public key (described in Section `ssh key generation`_) to the ``authorized_keys`` file in the containers as follows:

.. code-block:: bash

   $ docker exec -it openconfig1 sh
   # su - noc
   # echo "<root public key>" >> ~/.ssh/authorized_keys
   # chown noc:noc ~/.ssh/authorized_keys
   # chmod 400 ~/.ssh/authorized_keys

Repeat the steps for the second openconfig2 device.

Login
=====
You can now log in to the devices the addresses and your key using the NETCONF SSH subsystem as follows:

.. code-block:: bash

   $ sudo ssh noc@<IP address from above> -o StrictHostKeyChecking=no -s netconf
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability><capability>urn:ietf:params:netconf:base:1.0</capability><capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=0</capability><capability>urn:ietf:params:netconf:capability:candidate:1.0</capability><capability>urn:ietf:params:netconf:capability:validate:1.1</capability><capability>urn:ietf:params:netconf:capability:startup:1.0</capability><capability>urn:ietf:params:netconf:capability:xpath:1.0</capability><capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all,trim,report-all-tagged</capability><capability>urn:ietf:params:netconf:capability:notification:1.0</capability><capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring</capability></capabilities><session-id>2</session-id></hello>]]>]]>

Repeat the steps for the openconfig2 device.

Known hosts
-----------
The reason for this initial login is to setup "known hosts" of the controller. The controller refuses to open a connection to a device if the key to the device is not known.

However, you can solve the known hosts issue in several ways, including:

1. Remove the ``-o StrictHostKeyChecking=no`` and instead answer ``yes`` to the interactive prompt to add key to known hosts.
2. Edit the known-hosts file directly.
3. You can remove the requirement altoghether by configuring as follows::

    cli# set devices device openconfig1 ssh-strictkey false
    cli# commit

It is not recommended to remove the requirement, but may be necessary
in some circumstances, such as the existence of jump hosts.
  
Connect to devices
==================
To connect to the devices frm the controller, start the controller CLI and configure both devices added in previously:

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

Finally, connect to the devices. The expected result is that both devices are OPEN without log messages:

.. code-block:: bash

   user@test> connection open
   user@test> show connections
   Name                    State      Time                   Logmsg
   ================================================================
   openconfig1             OPEN       2024-09-02T14:15:59
   openconfig2             OPEN       2024-09-02T14:15:59

Both devices are now connected to the controller.

Errors
------
If a device fails when connecting, then a log error is displayed:

.. code-block:: bash

   cli>show connections
   Name                    State      Time                   Logmsg                        
   =======================================================================================
   openconfig1             CLOSED     2024-09-02T11:12:29    Closed by device
   openconfig2             OPEN       2024-09-02T14:15:59


The controller may be unable to login to the device for one of the following reasons:

   * The device has no NETCONF SSH subsystem enabled
   * The controllers public SSH key is not installed on the device
   * The device host key is not installed in the controllers `known_hosts`

The controller requires its public key to be installed on the devices and performs strict checking of host keys to avoid man-in-the-middle attacks. You need to ensure that the public key the controller uses is installed on the devices, and that the known_hosts file of the controller contains entries for the devices.

Next step
=========
Next step is either to continue with the :ref:`CLI tutorial <controller_cli>` or to start developing services using the :ref:`Service tutorial <tutorial>`.
