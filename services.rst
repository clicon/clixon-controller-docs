.. _controller_services:
.. sectnum::
   :start: 12
   :depth: 3

*******************
Service development
*******************

Service modules contains the actual code and logic which is used when
modifying the configuration three for services. 

The Python server looks for modules in the directory
``/usr/local/clixcon/controller/modules`` unless anything else is
defined) and when a module is launched by the Python server the server
call the setup method which must be present in every service module.

There must also be a variable named SERVICE which value should be the
same as the services name.

A minimal service module may look like this:

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      log.info("I am a module")


Module installation
===================

Clixon controller installs a utility named "clixon_controller_packages.sh"
in "/usr/local/bin". This can be used to install packages.

.. code:: bash

  $ clixon_controller_packages.sh -h
  Usage: /usr/local/bin/clixon_controller_packages.sh [OPTIONS]
    -s Source path
    -m Clixon controller modules install path
    -y Clixon controller YANG install path
    -r Use with care: Reset Clixon controller modules and YANG paths
    -h help


The script copies Python code and module YANG files
to the correct directories and take care of permissions etc.

The normal use case is to run the "clixon_controller_packages.sh" without
the ``-m`` and ``-y`` arguments, the script installs modules and YANG 
in the default paths which is preferred.

Modules basics
==============
The setup method take three parameters, root, log and kwargs. 

* `Root` is the configuration three.
* `Log` is used for logging and is a reference to a Python logging object. The log parameter can be used to print log messages. If the server is running in the foreground the log messages can be seen in the terminal, otherwise they will be written to syslog.
* `kwargs` is a dict of optional arguments. kwargs can contain the argument "instance" which is the name of the current service instance that is being changed by the user.

There is also a variable named "SERVICE" that should have the same name as the 
service without revision.

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      log.info("Informative log")
      log.error("Error log")
      log.debug("Debug log")

The root parameter is the configuration three received from the Clixon
backend.

Contents of the root parameter can be written in XML format by using the dumps() method:

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      log.debug(root.dumps())


Commit hooks
============

It is possible add "commit hooks" to a service module. A commit hook
is a method that is called when a "commit" or a "commit diff" is
issued. There are three types of commit hooks:

* setup_pre_commit, called before the commit is issued.
* setup_post_commit, called after the commit is issued.
* setup_post_commit_failed, called if the commit fails.

The commit hooks are optional and can be used to perform actions
before or after a commit. The commit hooks are defined in the service
module and must be named as described above.

The commit hooks take the same parameters as the setup method,
example:

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      log.info("I am a module")

  def setup_pre_commit(root, log, **kwargs):
      log.info("I am a pre commit hook")

  def setup_post_commit(root, log, **kwargs):
      log.info("I am a post commit hook")

  def setup_post_commit_failed(root, log, **kwargs):
      log.info("I am a post commit failed hook")


 The attribute kwargs will contain the argument "instance" which is
 the name of the current service instance that is being changed by the
 user and can also contain the argument "diff" which is a boolean
 value that is True if the commit is a "commit diff" and False if it
 is a "commit".

Service attributes
==================

When creating new nodes related to services it is important to append the proper
attributes to the new node. The Clixon backend will keep track of which nodes 
belongs to which service using the attribute cl:creator where the value of 
cl:create is the service name.

Example:

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      device.config.configuration.system.create("test", cdata="foo", 
      			attributes={"cl:creator": "test-service"})

Python object tree
==================

Manipulating the configuration tree is the central part of the
service modules. For example, a service could be defined with the only
purpose to change the hostname on devices.

In the Juniper CLI one would do something similar to this to configure
the hostname::

  admin@junos> configure
  Entering configuration mode

  [edit]
  admin@junos# set system host-name foo-bar-baz

  [edit]
  admin@junos# commit
  commit complete

However, in the Clixon CLI this behaviour can be modelled 
by using a service YANG models. For example, altering the
hostname for a lot of devices could look as follows::

  test@test> configure
  test@test[/]# set services hostname test hostname foo-bar-baz
  test@test[/]# commit

Clixon itself can not modify the configuration when the commit is
issued, but this must be implemented using a service module.

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      hostname = root.services.hostname.hostname

      for device in root.devices:
	  device.config.configuration.system.host_name

When the service module above is executed Clixon automatically calls
the setup method.

The "root" object is modified and passed as a parameter to setup. It
is parsed by the Python API and converted to a tree of Python objects.

One can also create new configurations. For example, the same example can be modified to
create a new node named test:

.. code:: python

  SERVICE = "example"


  def setup(root, log, **kwargs):
      device.config.configuration.system.create("test", cdata="foo")

The code above would translate to an NETCONF/XML string which looks like this:

.. code:: xml

  <device>
    <config>
      <configuration>
	<system>
	  <test>
	    foo
	  </test>
	</system>
      </configuration>
    </config>
  </device>

Object tree API
===============

Clixon Python API contains a few methods to work with the
configuration three.

Parsing
-------

The most fundamental method is parse_string from parse.py, this method
take any XML string and convert it to a tree of Python objects:

.. code:: python

  >>> from clixon.parser import parse_string
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>> root.xml.tags.tag
  foo
  >>>

As seen in the example above an object (root) is returned from
parse_string, root is a representation of the XML string xmlstr.

Something worth noting is that XML tags with '-' in them must be
renamed. A tag named "foo-bar" will have the name "foo_bar" after
being parsed since Python don't allow '-' in object names.

The original name is saved and when the object tree is converted back
to XML the original name is be present:

.. code:: python

  >>> xmlstr = "<xml><tags><foo-bar>foo</foo-bar></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>> root.xml.tags.foo_bar
  foo
  >>> root.dumps()
  '<xml><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

Creation
--------

It is also possible to create the tree manually:

.. code:: python

  >>> from clixon.element import Element
  >>>
  >>> root = Element("root")
  >>> root.create("xml")
  >>> root.xml.create("tags")
  >>> root.xml.tags.create("foo-bar", cdata="foo")
  >>> root.dumps()
  '<xml><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

Attributes
----------

For any object it is possible to add attributes:

.. code:: python

  >>> root.xml.attributes = {"foo": "bar"}
  >>> root.dumps()
  '<xml foo="bar"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>> root.xml.attributes["baz"] = "baz"
  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>>

The Python API is not aware of namespaces etc but the user must handle
that.

Adding tags
-----------

A new tag can now be added to root and look at the generated XML using
the method dumps():

.. code:: python

  >>> root.xml.create("foo", cdata="bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags><foo>bar</foo></xml>'
  >>>

Renaming tags
-------------

If needed the tag can be renamed:

.. code:: python

  >>> root.xml.foo.rename("bar", "bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags><bar>bar</bar></xml>'
  >>>

Removing tags
-------------

And remove the tag:

.. code:: python

  >>> root.xml.delete("bar")
  >>> root.dumps()
  '<xml><tags><tag>foo</tag></tags></xml>'
  >>>

Altering CDATA
--------------

CDATA can be altered:

.. code:: python

  >>> root.xml.tags.tag
  foo
  >>> root.xml.tags.tag.cdata = "baz"
  >>> root.xml.tags.tag
  baz
  >>> root.dumps()
  '<xml><tags><tag>baz</tag></tags></xml>'
  >>>

Iterate objects
---------------

We can also iterate over objects using tags:

.. code:: python

  >>> from clixon.parser import parse_string
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag><tag>bar</tag><tag>baz</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>>
  >>> for tag in root.xml.tags.tag:
  ...     print(tag)
  ...
  foo
  bar
  baz
  >>>
  >>> xmlstr = "<xml><tags><tag>foo</tag></tags></xml>"
  >>> root = parse_string(xmlstr)
  >>>
  >>> for tag in root.xml.tags.tag:
  ...     print(tag)
  ...
  foo

As seen above, there is a an XML string with a list of tags that can be iterated.

Adding objects
--------------

Objects can also be added to the tree:

.. code:: python

  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar></tags></xml>'
  >>> new_tag = Element("new-tag")
  >>> new_tag.create("new-tag")
  >>> root.xml.tags.add(new_tag)
  >>> root.dumps()
  '<xml foo="bar" baz="baz"><tags><foo-bar>foo</foo-bar><new-tag><new-tag/></new-tag></tags></xml>'
  >>>

The method add() adds the object to the tree and. The object must be
an Element object.
