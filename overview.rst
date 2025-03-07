.. _controller_overview:
.. sectnum::
   :start: 1
   :depth: 3

********
Overview
********

The Clixon controller is an open-source controller for network devices with a NETCONF/YANG API.

Its aim is to provide centralized automation of device operation using an interactive CLI and a Python engine over a network.

The controller sets up connections to devices, and controls them by monitoring their status and automate their configuration.

It supports multiple devices with different YANG models.

The controller is based on `Clixon <https://clixon-docs.readthedocs.io>`_.

Goals
-----
The Clixon network controller aims at providing a simple
network controller for NETCONF devices of different vendors.

Further goals are:

- Programmable network services, with a Python API
- Multiple devices, with different YANG schemas using `RFC 8528: YANG Schema Mount <http://www.rfc-editor.org/rfc/rfc8528.txt>`_ .
- Transactions with validate/commit/revert across groups of devices
- Scaling up to 100 devices.

What is NETCONF/YANG?
---------------------
NETCONF is a network management protocol providing a transactional semantics (commit/rollback).
YANG is a data modeling language for the definition of data sent over (for example) NETCONF.
NETCONF and YANG are IETF standards and a growing number of devices provide APIs.

Architecture
------------
.. image:: controller.jpg
   :width: 100%

The controller is built on the base of the `CLIgen/Clixon <https://clicon.org>`_ system, where
the controller semantics is implemented using plugins. The `backend`
is the core of the system controlling the datastores and accessing the
YANG models.

APIs
----
The `southbound API` uses only NETCONF over SSH to network
devices. There are no current plans to support other protocols for
device control.

The `northbound APIs` are YANG-derived Restconf, Autocli, Netconf, and
Snmp.  The controller CLI has two modes: operation and configure, with
an autocli configure mode derived from YANG.

A PyAPI module accesses configuration data via the `actions API <controller_actions>`_. The
PyAPI module reads services configuration and writes device data. The
backend then pushes changes to the actual devices using a transaction
mechanism.
