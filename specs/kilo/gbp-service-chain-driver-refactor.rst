..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Service Chain Driver Refactor
==========================================


Problem description
===================
Current service chain driver is a monolithic entity that couples the service
chaining logic along with the service configuration logic. Decoupling of these
entities will allow development of a service configuration driver independent
of the chaining mechanism.

Proposed change
===============
At a high level the following changes are proposed:

1. Refactor current Service Chain Service structure in order to
   easily accommodate new plugins;

2. Create a new API object called "Service Profile". This object
   contains a set of attributes that can describe the service
   (eg. service_type, vendor, insertion_mode and so forth).
   Service Profile will be extensible from day one;

3. Replace the "service_type" attribute on the Service Chain Node
   with the "service_profile" attribute. The latter is an UUID
   pointing to an existing Service Profile object;

4. Create a new "Node Composition Plugin". The Plugin can load one or
   multiple "Node Driver(s)". A Node Driver is capable of deploying,
   destroying and updating Service Chain Node instances depending
   on their profile;

5. Define Service Configuration and Management driver interface;

6. Implement 2 reference implementations of Node Drivers.
   They will use Nova (for NFV) and Neutron in the backend.

The relationship between the Services Plugin and Node Drivers is as shown below:


The Node Centric Chain Plugin  implementation is designed as the following class
hierarchy:

asciiflow::

 +--------------------------------------+              +---------------------------+
 |NodeComposPlugin(ServiceChainDbPlugin)|              |      NodeDriverBase       |
 |                                      |              |                           |
 |                                      |              |                           |
 |                                      |              |                           |
 |                                      |              |                           |
 |                                      |              |                           |
 |                                      |              |                           |
 |                                      |1            N|                           |
 |                                      +--------------+                           |
 +--------------------------------------+              +---------------------------+
 | *create       *update      *delete   |              | *plumbing_info()          |
 |    *SCI          *SCI         *SCI   |              | *validate(SCN)            |
 |    *SCS          *SCS         *SCS   |              | *deploy(NodeContext)      |
 |    *SCN          *SCN         *SCN   |              | *destroy(NodeContext)     |
 +--------------------------------------+              | *update(NodeContext)      |
                                                       +----^--------^--------^----+
 +--------------------------------------+                   |        |        |
 |            NodeContext               |                   |        |        |
 |                                      |          +--------+----+   |   +----+--------+
 | *core plugin                         |          |             |   |   |             |
 | *sc plugin                           |          | Nova        |   |   | Neutron     |
 | *provider ptg                        |          | Node        |   |   | Node        |
 | *consumer ptg                        |          | Driver      |   |   | Driver      |
 | *policy target(s)                    |          |             |   |   |             |
 | *management ptg                      |          |             |   |   |             |
 | *ser^ice chain instance              |          +-----------^-+   |   +-^-----------+
 | *service chain node                  |                      |     |     |
 | *service chain spec                  |                      |     |     |
 +--------------------------------------+                  +---+-----+-----+---+
 |                                      |                  |                   |
 | *get/update/delete service targets   |                  |  SC Node Driver   |
 |                                      |                  |                   |
 +--------------------------------------+                  +-------------------+


Node Driver Base
This supports operations for CRUD of a service, and to query the number of
data-path and management interfaces required for this service.

Node Context
Provides useful attributes and methods for the Node Driver to use.
CRUD on "service targets" are useful to create service specific
Policy Targets in defined PTGs (provider/consumer/management)

The Node Driver operations are called as pre-/post-commit hooks.

Service Targets
This is an *internal only* construct. It's basically a normal Policy Target
but with some metadata which makes easy to understand which service it
belongs to, in which order, on which side of the relationship, for which
Node, deployed by which driver. Will require a new table to store all
these info.

Nova Node Driver
This provides a reusable implementation for managing the lifecycle of a
service VM.

Neutron Node Driver
This provides a reusable implementation for managing existing Neutron
services.

Node Driver
This configures the service based on the “config” provided in the Service
Node definition.

Data model impact
-----------------

Service Target
  * policy_target_id - PT UUID
  * service_chain_instance_id - SCI UUID
  * service_chain_node_id - SCN UUID, the one of the specific node this ST belongs to
  * relationship - Enum, PROVIDER|CONSUMER|MANAGEMENT
  * order - Int, order of the node within the chain

Service Profile
  * id - standard object uuid
  * name - optional name
  * description - optional annotation
  * shared - whether the object is shared or not
  * vendor - optional string indicating the vendor
  * insertion_mode - string GATEWAY|ENDPOINT|TRANSPARENT
  * service_type -  generic string (eg. LOADBALANCER|FIREWALL|...)

Service Chain Node
  * REMOVE service_type
  * service_profile_id - SP UUID

REST API impact
---------------

The REST API changes look like follows::

 SERVICE_PROFILES: {
     'id': {'allow_post': False, 'allow_put': False,
            'validate': {'type:uuid': None}, 'is_visible': True,
            'primary_key': True},
     'name': {'allow_post': True, 'allow_put': True,
              'validate': {'type:string': None},
              'default': '', 'is_visible': True},
     'description': {'allow_post': True, 'allow_put': True,
                     'validate': {'type:string': None},
                     'is_visible': True, 'default': ''},
     'tenant_id': {'allow_post': True, 'allow_put': False,
                   'validate': {'type:string': None},
                   'required_by_policy': True, 'is_visible': True},
     attr.SHARED: {'allow_post': True, 'allow_put': True,
                   'default': False, 'convert_to': attr.convert_to_boolean,
                   'is_visible': True, 'required_by_policy': True,
                   'enforce_policy': True},
     'vendor': {'allow_post': True, 'allow_put': True,
                'validate': {'type:string': None},
                'is_visible': True, 'default': ''},
     'insertion_mode': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:values':
                                     scc.VALID_INSERTION_MODES},
                        'is_visible': True, 'default': None},
     'service_type': {'allow_post': True, 'allow_put': True,
                      'validate': {'type:string': None},
                      'is_visible': True, 'required': True},
 }

The following is added to servicechain node::

 SERVICECHAIN_NODES: {
      'service_profile_id': {'allow_post': True, 'allow_put': True,
                             'validate': {'type:uuid': None},
                             'required': True, 'is_visible': True},
  }

Security impact
---------------


Notifications impact
--------------------


Other end user impact
---------------------


Performance impact
------------------


Other deployer impact
---------------------

TBD

Developer impact
----------------

TBD

Community impact
----------------


Alternatives
------------


Implementation
==============

Assignee(s)
-----------

* Ivar Lazzaro (mmaleckk)

Work items
----------


Dependencies
============


Testing
=======

Tempest tests
-------------


Functional tests
----------------


API tests
---------


Documentation impact
====================

User documentation
------------------


Developer documentation
-----------------------


References
==========


