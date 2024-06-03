.. _controller_quickstart:
.. sectnum::
   :start: 3
   :depth: 3

***********
Quick start
***********

Start contreoller and two example openconfig devices as docker containers. It requires docker compose and you may need to be sudo::

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

