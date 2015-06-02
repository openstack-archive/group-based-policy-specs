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

4.1. The plumbing info of all the scheduled nodes will be used by the
     NCP for traffic stitching/steering. This will be a pluggable module
     (NodePlumber).

5. Define Service Configuration and Management driver interface;

6. Implement 2 reference implementations of Node Drivers.
   They will use Nova (for NFV) and Neutron in the backend.

The relationship between the Services Plugin and Node Drivers is as shown below:


The Node Composition Plugin  implementation is designed as the following class
hierarchy:

asciiflow::

 +--------------------------------------+      +-----------------------------------------+
 |NodeComposPlugin(ServiceChainDbPlugin)|      |      NodeDriverBase                     |
 |                                      |      |                                         |
 |                                      |      |                                         |
 |                                      |      |                                         |
 |                                      |      |                                         |
 |                                      |      |                                         |
 |                                      |      |                                         |
 |                                      |1    N|                                         |
 |                                      +------+                                         |
 +--------------------------------------+      +-----------------------------------------+
 | *create       *update      *delete   |      | *get_plumbing_info()                    |
 |    *SCI          *SCI         *SCI   |      | *validate(NodeContext)                  |
 |    *SCS          *SCS         *SCS   |      | *create(NodeContext)                    |
 |    *SCN          *SCN         *SCN   |      | *delete(NodeContext)                    |
 +-----------------+--------------------+      | *update(NodeContext)                    |
                   |                           | *update_policy_target_added(NContext,PT)|
 +-----------------+--------------------+      | *update_policy_target_removed(...)      |
 |NodePlumber                           |      |                                         |
 |                                      |      |                                         |
 |                                      |      |                                         |
 +--------------------------------------+      |                                         |
 |                                      |      +---------^----------^----------^---------+
 | *plug_services(NContext,Deployment)  |                |          |          |
 | *unplug_services(NContext,Deployment)|                |          |          |
 |                                      |         +------+------+   |   +------+------+
 +--------------------------------------+         |             |   |   |             |
                                                  | Nova        |   |   | Neutron     |
 +--------------------------------------+         | Node        |   |   | Node        |
 |            NodeContext               |         | Driver      |   |   | Driver      |
 |                                      |         |             |   |   |             |
 | *core plugin                         |         |             |   |   |             |
 | *sc plugin                           |         +-----^-------+   |   +------^------+
 | *provider ptg                        |               |           |          |
 | *consumer ptg                        |               |           |          |
 | *policy target(s)                    |               |           |          |
 | *management ptg                      |         +-----+----+ +----+---+ +----+-----+
 | *service chain instance              |         | SC Node  | | SC Node| | SC Node  |
 | *service chain node                  |         | Driver   | | Driver | | Driver   |
 | *service chain spec                  |         +----------+ +--------+ +----------+
 +--------------------------------------+
 |                                      |
 | *get/update/delete service targets   |
 |                                      |
 +--------------------------------------+


Node Driver Base
This supports operations for CRUD of a service, and to query the number of
data-path and management interfaces required for this service.
Also supports call backs for auto-scaling in case of added/removed PTs
on a relevant PTG.

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

Node Plumber
The node plumber is an entity which takes care of plumbing nodes in a
chain. By node plumbing is intended the creation/disruption of the
appropriate Neutron and GBP constructs (typically Ports and Policy Targets)
based on the specific Node needs, taking into account the whole service
chain in the process. Ideally, this module will ensure that the traffic
flows as expected according to the user intent.

Deployment (input parameter in plug and unplug services methods)
A deployment is a list composed as follows::

 [{'context': node_context,
  'driver': deploying_driver,
  'plumbing_info': node_plumbing_needs},
   ...]

No assumptions should be made on the order of the nodes as received in
the deployment, but it can be retrieved by calling node_context.current_position


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
  * insertion_mode - string L2|L3|BITW|TAP
  * service_type -  generic string (eg. LOADBALANCER|FIREWALL|...)
  * service_flavor - generic string (eg. m1.tiny)

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
     'service_flavor': {'allow_post': True, 'allow_put': True,
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


