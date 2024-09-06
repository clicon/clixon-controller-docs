.. _setup_tutorial:
.. sectnum::
   :start: 4
   :depth: 3

**************
Setup tutorial
**************

This tutorial to setup a controller environment on a Linux machine.

The aim is to setup a controller with two virtual openconfig devices.

You need such a setup if you want to adapt the controller to your own devices or develop a service.

You can skip this tutorial if you only want to run the controller CLI using the docker demo.

Prerequisites
=============

Before you start, you need to have the following:

1. Either a virtual machine or a physical machine where you have root
   access (the host)
2. Docker installed on the machine.

Environment
-----------
The environment consists of one controller and two OpenConfig
devices. The controller communicates with the devices using NETCONF
tunneled over SSH and the user interacts with the controller using
a CLI.

The controller runs on the host machine and the devices are
running in Docker containers. The devices are a dockerized installation of
Clixon with the OpenConfig models.

Controller installation
-----------------------

See the :ref:`Installation <controller_install>` section for more
information on how to set up the controller.

SSH key generation
------------------

To be able to log in to the devices, we need to generate an SSH key
pair if you don't already have one. The public key will be added to
the devices. To see if you have an SSH key pair, check the contents of
the .ssh directory in your root directory:

.. code-block:: bash

   $ sudo ls /root/.ssh
   authorized_keys  id_rsa  id_rsa.pub  known_hosts

If you don't have an SSH key pair, generate one using the following
command:

.. code-block:: bash

   $ sudo ssh-keygen -C "controller key" -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""

To get your public key which we will use later, run the following
command:

.. code-block:: bash

   $ sudo cat /root/.ssh/id_rsa.pub

Docker setup
------------

The devices are running in Docker containers. The Docker image is a
pre-built image with the OpenConfig models. The image is available on
Docker Hub and can be pulled using the following command:

.. code-block:: bash

   $ docker pull clixon/openconfig

To start the devices, run the following commands:

.. code-block:: bash

   $ docker run -d --name openconfig1 -it clixon/openconfig
   $ docker run -d --name openconfig2 -it clixon/openconfig

The devices should then be visible in the list of running containers:

.. code-block:: bash

   $ docker ps
   CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                      NAMES
   1b3b3b3b3b3b        clixon/openconfig    "/bin/sh"           2 minutes ago       Up 2 minutes       22/tcp, 80/tcp, 830/tcp    openconfig1
   2a2a2a2a2a2a        clixon/openconfig    "/bin/sh"           2 minutes ago       Up 2 minutes       22/tcp, 80/tcp, 830/tcp    openconfig2

The devices are now running and we can connect to them using SSH. In
order to do that, we need to add our public key to the devices. Add
your local root users key to the authorized_keys file in the device:

.. code-block:: bash

   $ docker exec -it openconfig1 sh
   # su - noc
   # echo "<your public key>" >> ~/.ssh/authorized_keys
   # chown noc:noc ~/.ssh/authorized_keys
   # chmod 400 ~/.ssh/authorized_keys

Repeat the steps for the second device which is named
openconfig2. Then verify that you can log in to the devices using your
key and the NETCONF subsystem is running. To log in using SSH we
should first find the IP addresses of the devices:

.. code-block:: bash

   $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' openconfig1
   $ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' openconfig2

The addresses are for example 172.17.0.2 and 172.17.0.3, use them to
log in to the devices with the user noc and the NETCONF subsystem:

.. code-block:: bash

   $ sudo ssh noc@<IP address from above> -s netconf
   <hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0"><capabilities><capability>urn:ietf:params:netconf:base:1.1</capability><capability>urn:ietf:params:netconf:base:1.0</capability><capability>urn:ietf:params:netconf:capability:yang-library:1.0?revision=2019-01-04&amp;module-set-id=0</capability><capability>urn:ietf:params:netconf:capability:candidate:1.0</capability><capability>urn:ietf:params:netconf:capability:validate:1.1</capability><capability>urn:ietf:params:netconf:capability:startup:1.0</capability><capability>urn:ietf:params:netconf:capability:xpath:1.0</capability><capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&amp;also-supported=report-all,trim,report-all-tagged</capability><capability>urn:ietf:params:netconf:capability:notification:1.0</capability><capability>urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring</capability></capabilities><session-id>2</session-id></hello>]]>]]>

Repeat the steps for the second device and name it openconfig2.

Now we have to working OpenConfig devices which we can connect to
using NETCONF tunneled over SSH. The next step is to add the devices
to the controller. This is done from the CLI:

Add devices to the controller
=============================

To add the devices to the controller, start the CLI and configure both
of the devices added in the previous step:

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

And then connect to the devices, we expect the connection state to be
OPEN for both devices and no log messages:

.. code-block:: bash

   user@test> connection open
   user@test> show connections
   Name                    State      Time                   Logmsg
   ================================================================
   openconfig1             OPEN       2024-09-02T14:15:59
   openconfig2             OPEN       2024-09-02T14:15:59

Both devices are now connected to the controller and we can start
working with the service.
