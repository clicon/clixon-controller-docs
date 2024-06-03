.. _controller_quickstart:
.. sectnum::
   :start: 3
   :depth: 3

***********
Quick start
***********

Start example devices as containers::

  cd test
  ./start-devices.sh
  sudo ./copy-keys.sh

Start controller::

  sudo clixon_backend -f /usr/local/etc/clixon/controller.xml

Start the CLI and configure devices::

  clixon_cli -f /usr/local/etc/controller.xml -m configure
  set devices device clixon-example1 description "Clixon container"
  set devices device clixon-example1 conn-type NETCONF_SSH
  set devices device clixon-example1 addr 172.20.20.2
  set devices device clixon-example1 user root
  set devices device clixon-example1 enable true
  set devices device clixon-example1 yang-config VALIDATE
  set devices device clixon-example1 root
  commit local

Thereafter explicitly connect to the devices::

  clixon_cli -f /usr/local/etc/clixon/controller.xml
  connection open
