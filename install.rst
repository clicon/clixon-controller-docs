.. _controller_install:
.. sectnum::
   :start: 2
   :depth: 3

************
Installation
************

Packages
--------
Some packages are required. The following are example of debian-based packages::
  
  sudo apt install flex bison git make gcc libnghttp2-dev libssl-dev

  
Source
------
Check out the following GIT repos:

- `<https://github.com/clicon/cligen.git/>`_
- `<https://github.com/clicon/clixon.git/>`_
- `<https://github.com/clicon/clixon-controller.git/>`_
- `<https://github.com/clicon/clixon-pyapi.git/>`_

Building
--------
The source is built as follows.

Cligen
^^^^^^
::

  cd cligen
  ./configure
  make
  sudo make install

Clixon
^^^^^^
::
   
  cd clixon
  ./configure
  make
  sudo make install

Python API
^^^^^^^^^^
::

  # Build and install the package
  cd clixon-pyapi
  sudo -u clicon pip3 install -r requirements.txt
  sudo python3 setup.py install
  
Controller
^^^^^^^^^^
::
   
  cd clixon-controller
  ./configure
  make
  sudo make install
  sudo mkdir /usr/local/share/clixon/controller/mounts/

Install
-------
Install the python code by copy::

  sudo cp clixon_server.py /usr/local/bin/

Add a new clicon user and install the needed Python packages,
the backend will start the Python server and drop the privileges
to this user::

  sudo useradd -g clicon -m clicon
