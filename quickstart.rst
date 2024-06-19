.. _controller_quickstart:
.. sectnum::
   :start: 3
   :depth: 3

***********
Quick start
***********

Docker compose
==============

Start controller and two example openconfig devices as docker containers. It requires docker compose and you may need to be sudo. This is how regression tests run::

  $ ./start-demo-containers.sh
  $ docker exec -it demo-controller clixon_cli
  nobody@3e29b6e15c34>
  nobody@0b2157135dee> connection open
  nobody@0b2157135dee> show connections
  Name                    State      Time                   Logmsg
  ==================================================================
  openconfig1             OPEN       2024-06-03T13:13:49
  openconfig2             OPEN       2024-06-03T13:13:49
  nobody@0b2157135dee>

CLI setup
=========

Start the controller and setup devices by editing using the CLI

Controller
----------
Start the controller manually (or via systemd)::

  clixon_backend -f /usr/local/etc/clixon/controller.xml

Start devices and ensure reachability via SSH netconf subsystem.

Start the CLI and set up devices::

  clixon_cli -f /usr/local/etc/clixon/controller.xml -l s
  cli> configure
  cli# edit devices device mydevice
  cli[device=mydevice]# set addr 172.17.0.3
  cli[device=mydevice]# set user admin
  cli[device=mydevice]# set conn-type NETCONF-SSH
  cli[device=mydevice]# commit local
  cli[device=mydevice]# exit
  cli> connection open
  cli> show connections
  Name                 State      Time                   Logmsg
  ==================================================================
  mydevice             OPEN       2024-06-03T13:13:49
