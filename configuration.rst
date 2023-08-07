.. _controller_configuration:
.. sectnum::
   :start: 4
   :depth: 3

*************
Configuration
*************

The controller extends the clixon configuration file as follows:

``CLICON_CONFIG_EXTEND``
   The value should be `clixon-controller-config` making the controller-specific 

``CONTROLLER_ACTION_COMMAND``
   Should be set to the PyAPI binary with correct arguments
   The namespace is ="http://clicon.org/controller-config"

``CLICON_BACKEND_USER``
   Set to the user which the action binary (above) is used. Normally `clicon`

``CLICON_SOCK_GROUP``   
   Set to user group, ususally `clicon`

``CONTROLLER_PYAPI_MODULE_PATH``
   Path to Python code for PyAPI
   
``CONTROLLER_PYAPI_MODULE_FILTER``

``CONTROLLER_PYAPI_PIDFILE``
   
Example
-------
The following configuration file examplifies the configure options described above::

  <clixon-config xmlns="http://clicon.org/config">
  <CLICON_CONFIGFILE>/usr/local/etc/controller.xml</CLICON_CONFIGFILE>
  <CLICON_FEATURE>ietf-netconf:startup</CLICON_FEATURE>
  <CLICON_FEATURE>clixon-restconf:allow-auth-none</CLICON_FEATURE>
  <CLICON_CONFIG_EXTEND>clixon-controller-config</CLICON_CONFIG_EXTEND>
  <CONTROLLER_ACTION_COMMAND xmlns="http://clicon.org/controller-config">
        /usr/local/bin/clixon_server.py -f /usr/local/etc/controller.xml -F
  </CONTROLLER_ACTION_COMMAND>
  <CONTROLLER_PYAPI_MODULE_PATH xmlns="http://clicon.org/controller-config">
        /usr/local/share/clixon/controller/modules/
  </CONTROLLER_PYAPI_MODULE_PATH>
  <CONTROLLER_PYAPI_MODULE_FILTER xmlns="http://clicon.org/controller-config"></CONTROLLER_PYAPI_MODULE_FILTER>
  <CONTROLLER_PYAPI_PIDFILE xmlns="http://clicon.org/controller-config">
        /tmp/clixon_server.pid
  </CONTROLLER_PYAPI_PIDFILE>
  <CLICON_BACKEND_USER>clicon</CLICON_BACKEND_USER>
  <CLICON_SOCK_GROUP>clicon</CLICON_SOCK_GROUP>
