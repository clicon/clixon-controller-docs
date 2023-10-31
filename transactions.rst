.. _controller_transactions:
.. sectnum::
   :start: 7
   :depth: 3
   
************
Transactions
************

A basis of controller operation is the use of transactions. Clixon itself has underlying candidate/running datastore transactions. The controller expands the transaction concept to span multiple devices.
There are two such types of composite transactions:

1. `Device connect`: where devices are connected via NETCONF over ssh, key exchange, YANG retrieval and config pull
2. `Config push`: where a service is (optionally) edited, changed device config is pushed to remote devices via NETCONF.

.. image:: transaction.jpg
   :width: 100%


Device connect
==============

A `device connect` transaction starts in state `CLOSED` and if succesful stops in `OPEN`. there are multiple intermediate steps as follows (for each device):

1. An SSH session is created to the IP address of the device
2. An SSH login is made which requires:

   a) The device to have enabled a NETCONF ssh sub-system
   b) The public key of the controller to be installed on the device
   c) The public key of the device to be in the `known_hosts` file of the controller
3. A mutual NETCONF `<hello>` exchange
4. Get all YANG schema identifiers from the device using the ietf-netconf-monitoring schema.
5. For each YANG schema identifier, make a `<get-schema>` RPC call (unless already retrieved).
6. Get the full configuration of the device.

Config push
===========
While a `device connect` operates on individual devices, the `config push` transaction operates on all devices. It starts in `OPEN` for all devices and ends in `OPEN` for all devices involved in the transaction:

1. The user edits a service definition and commits
2. The commit triggers PyAPI services code, which rewrites the device config
3. Alternatively, the user edits the device configuration manually
4. The updated device config is validated by the controller
5. The remote device is checked for updates, if it is out of sync, the transaction is aborted
6. The new config is pushed to the remote devices
7. The new config is validated on the remote devices
8. If validation succeeds on all remote devices, the new config is committed to all devices
9. The new config is retreived from the device and is installed on the controller
10. If validation is not successful, or only a `push validate` was requested, the config is reverted on all remote devices.

Use the show transaction command to get details about transactions::

   cli> show transaction
     <transaction>
        <tid>2</tid>
        <state>DONE</state>
        <result>FAILED</result>
        <description>pull</description>
        <origin>example1</origin>
        <reason>validation failed</reason>
        <timestamp>2023-03-27T18:41:59.031690Z</timestamp>
     </transaction>

