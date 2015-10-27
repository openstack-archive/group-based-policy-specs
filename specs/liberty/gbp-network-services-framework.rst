..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================
Network Services Framework
===========================


Problem description
===================

GBP provides the mechanism to define a chain of network services [1] to process
the traffic between a pair of consumer and provider Policy Target Groups. The
network services include services such as Firewalls, Loadbalancers, VPN. The
network services could be inserted in different modes as addressed by the
Traffic Stitching Plumber specification [2][3]. In order to render policies that
include network services, GBP requires a framework to instantiate and manage the
network services.

Proposed change
===============

The proposal of the BP is to implement Network Services’s Framework (NSF) in GBP
project to handle lifecylce management of network services that includes
creation, deployment, management and resource pooling, monitoring capabilities
of network services. The proposal extends the GBP platform to provide these
capabilities that are common to all network services and is agnostic to the type
and implementation of the network service.

The Network Services’ Framework will adopt an extensible and pluggable driver
driven approach. The framework will define an API and resources that the GBP
Node Composition Plugin’s node drivers can make use of. It will also allow GBP
users to correlate logical Service Nodes with the actual service instances that
render them in a particular chain.

The current proposal is to implement a scalable RPC server to listen to
lifecycle and configuration events emitted by the node drivers and render the
configuration on the services being deployed or already deployed.

Data model impact
-----------------

The following resources will be used for the implementation:

1. ServiceInstance

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|provider_port      |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|consumer_port      |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_chain_id   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_profile_id |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_config     |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_vm_id      |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+

2. ServiceProfile

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_type       |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_vendor     |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_ha         |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_sharing    |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_image_id   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+

3. ServiceVM

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_port_id       |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_floating_ip   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|monitoring_port_id |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|reference_count    |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|interface_count    |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|status             |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+

REST API impact
---------------

The REST API to implement the CRUD for the resources will be implemented.

Security impact
---------------


Notifications impact
--------------------
The following RPC provide the Network Services Framework functionality:
* create_service
* get_services
* delete_service
* get_service_info

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

* Subrahmanyam Ongole (osms69)
* Magesh GV (magesh-gv)
* Rukhsana Ansari (rukansari)
* Hemanth Ravi (hemanth-ravi)
* Sumit Naiksatam (snaiksat)

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

[1] https://github.com/openstack/group-based-policy-specs/blob/master/specs/kilo/gbp-service-chain-driver-refactor.rst
[2] https://github.com/openstack/group-based-policy-specs/blob/master/specs/kilo/gbp-traffic-stitching-plumber.rst
[3] https://github.com/openstack/group-based-policy-specs/blob/master/specs/kilo/traffic-stitching-plumber-placement-type.rst
