..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Service Chain Driver Refactor
==========================================


Problem Description
===================
Current service chain driver is a monolithic entity that couples the service
chaining logic along with the service configuration logic. Decoupling of these
entities will allow development of a service configuration driver independent
of the chaining mechanism.

Proposed Change
===============
At a high level the following changes are proposed:

1. Refactor current Service Chain driver to remove service configuration
   specifics
2. Define Service Configuration and Management driver interface
3. Service Chain Plugin loads a single Service Chain Driver
4. Redefine “service_type” attribute of Service Node to be a key-value
   pair dict to be able to capture vendor, service class, etc.

The relationship between the Services’ Plugin, Service Chain Driver,
and Nodee Drivers is as shown below:

Service Chain Driver:
The Service Chain Driver implementation is designed as the following class
hierarchy:

asciiflow::

 +-----------------+
 |                 |
 |Services' Plugin |
 |                 |
 +-----------------+
         |1
         |
         |1
 +-------v---------+
 |                 |
 | Service Chain   |
 | Driver          |
 |                 |
 |                 |
 +-----------------+
         |1
         |
         |n
 +-------v---------+
 |                 |
 | Node Driver     |
 |                 |
 +-----------------+


Node Driver Base
This supports operations for CRUD of a service, and to query the number of
data-path and management interfaces required for this service.

asciiflow::

 +-----------------+
 |                 |
 |Node Driver Base |
 |                 |
 +-------/\---------+
         |          
         |          
         |          
 +-----------------+
 |                 |
 |                 |
 | VM Driver       |
 |                 |
 |                 |
 +-------/\---------+
         |          
         |          
         |          
 +-----------------+
 |                 |
 | Node Driver     |
 |                 |
 +-----------------+

The CU operations are passed the following context from the Service Chain
Driver:
provider_ptg
consumer_ptg
policy_target_name(s)
management_ptg
service_chain_instance_id
service_order

The Node Driver operations are called as pre-/post-commit hooks.

VM Driver Base
This provides a reusable implementation for managing the lifecycle of a
service VM.

Node Driver
This configures the service based on the “config” provided in the Service
Node definition.

Data Model Impact
-----------------
Changes pertaining to the use of service_type and service_meta_data


REST API Impact
---------------


Security Impact
---------------


Notifications Impact
--------------------


Other End User Impact
---------------------


Performance Impact
------------------


Other Deployer Impact
---------------------


Developer Impact
----------------


Community Impact
----------------


Alternatives
------------


Implementation
==============

Assignee(s)
-----------


Work Items
----------


Dependencies
============


Testing
=======

Tempest Tests
-------------


Functional Tests
----------------


API Tests
---------


Documentation Impact
====================

User Documentation
------------------


Developer Documentation
-----------------------


References
==========


