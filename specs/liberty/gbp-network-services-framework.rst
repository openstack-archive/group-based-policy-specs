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
the traffic between a provider and multiple consumer Policy Target Groups. The
network services include services such as Firewalls, Load Balancers, VPN. The
network services could be inserted in different modes as addressed by the
Traffic Stitching Plumber specification [2][3]. In order to render policies that
include network services, GBP requires a framework to instantiate and manage the
network services.

Proposed change
===============

The proposal of the BP is to implement Network Services’ Framework (NSF) in GBP
project to handle lifecycle management of network services that includes
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

The components of NSF and the interactions are depicted in the diagram below:

asciiflow::

   +-------------------+              +-----------------------+
   | Node Comp Plugin  |     +------->|      NSF Plugin       |
   +-------------------+     |        +-----------------------+
       +---------------+     |        +----------+
       |NSF Node Driver|-----+        |NSC Driver|
       +---------------+              +----------+
                                           |
                                           | RPC
                                           |
                                  +--------v---------+
                                  |                  |
                                  |  Network Service |
                  --------------->|     Controller   |
      Configuration Event RPC     |                  |
                                  +------------------+

NSF implementation will include the following components:

1. NSF Plugin to provide the CRUD APIs for the resources defined in the data
   model section. NSF plugin allows for the configuration of a driver to
   provide the implementation for the resources. NSF plugin interfaces with the
   drivers using pre/post commit hooks.
2. A reference Network Service Controller (NSC) Driver that interfaces with
   the reference Network Service Controller. The NSC driver can be replaced
   with another driver to customize the implementation.
3. A reference Network Service Controller implementation that implements the
   life-cycle management of Network Service VMs and renders the configuration
   of network services deployed on these VMs. The Network Service Controller uses
   RPC to communicate with the NSC driver and to listen for neutron configuration
   events.
4. NSF Node Driver which is a Node Composition Plugin driver to provision Service
   nodes using the NSF Plugin and render the configuration of Service nodes using
   neutron configuration APIs. Any Node driver running in the context of Node
   Composition Plugin can interface with NSF plugin to manage network services.

Data model impact
-----------------

The following resources will be used for the implementation:

1. NetworkService

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |name           |
+-------------------+--------+---------+----------+-------------+---------------+
|service_id         |UUID    |RW, all  |required  |             |GBP Service    |
|                   |        |         |          |             |Node Id or     |
|                   |        |         |          |             |Neutron        |
|                   |        |         |          |             |Service Id     |
+-------------------+--------+---------+----------+-------------+---------------+
|service_chain_id   |UUID    |RW, all  |          |             |GBP Service    |
|                   |        |         |          |             |Chain Instance |
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|service_profile_id |UUID    |RW, all  |          |             |Service Profile|
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|service_config     |string  |RW, all  |          |             |Device Specific|
|                   |        |         |          |             |Configuration  |
+-------------------+--------+---------+----------+-------------+---------------+
|data_port_pairs    |list    |RW, all  |required  |dictionary   |list of        |
|                   |(UUID)  |         |          |             |consumer,      |
|                   |        |         |          |             |provider port  |
|                   |        |         |          |             |id pairs       |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_ports         |list    |RO, all  |generated |             |list of mgmt   |
|                   |(UUID)  |         |          |             |port ids       |
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+

2. NetworkServiceInstance

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |name           |
+-------------------+--------+---------+----------+-------------+---------------+
|network_service_id |UUID    |RW, all  |required  |foreign-key  |NetworkService |
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|data_port_pair     |string  |RW, all  |required  |dictionary   |consumer,      |
|                   |        |         |          |             |provider port  |
|                   |        |         |          |             |id pair        |
+-------------------+--------+---------+----------+-------------+---------------+
|ha_state           |string  |RW, all  |''        |             |active or      |
|                   |        |         |          |             |standby HA mode|
+-------------------+--------+---------+----------+-------------+---------------+
|device_instance_id |UUID    |RW, all  |required  |foreign-key  |Id of device   |
|                   |        |         |          |             |deploying the  |
|                   |        |         |          |             |ServiceInstance|
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+

3. NetworkServiceDevice

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |name           |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_port_id       |UUID    |RW, all  |required  |             |management port|
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_floating_ip   |string  |RW, all  |required  |ip address   |management port|
|                   |        |         |          |             |floating ip    |
+-------------------+--------+---------+----------+-------------+---------------+
|monitoring_port_id |UUID    |RW, all  |          |             |monitoring port|
|name               |        |         |          |             |id for HA pair |
+-------------------+--------+---------+----------+-------------+---------------+
|service_vendor     |string  |RO, all  |          |             |vendor         |
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+

4. DevicePortContext

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|port_id            |string  |RW, all  |          |             |neutron port Id|
|                   |(UUID)  |         |          |  .          |               |
+-------------------+--------+---------+----------+-------------+---------------+
|pt_id              |UUID    |RW, all  |          |             |policy target  |
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|ptg_id             |UUID    |RW, all  |          |             |policy target  |
|                   |        |         |          |             |group Id       |
+-------------------+--------+---------+----------+-------------+---------------+
|ns_device_id  	    |UUID    |RW, all  |          |foreign-key  |NetworkService |
|                   |        |         |          |             |Device Id      |
+-------------------+--------+---------+----------+-------------+---------------+

Usage:

1. The NetworkService resource is created by the users of NSF, for e.g. the NSF
   Node driver and the other resources are created internally for e.g. by the
   Network Service Controller
2. All resources are available to be read using the REST API.
3. The CLI to create NetworkService would be as following:

        gbp networkservice-create --service  sn_1 --service_chain sci_10 --service_profile sp_1 --data_port_pairs "consumer=port_1,provider=port_2,type=active" --data_port_pairs "consumer=port_3,provider=port4,type=passive" --data_port_pairs "consumer=port_1,provider=port_2,type=vip" ns_1

4. The other resources can be displayed as following:

        gbp networkserviceinstance-list
        gbp networkserviceinstance-show si_1

        gbp deviceinstance-list
        gbp deviceinstance_show di_1


REST API impact
---------------

The REST API to implement the CRUD for the resources will be implemented.

Security impact
---------------


Notifications impact
--------------------
The following RPC provide the Network Services Framework functionality:
* create_networkservice
* get_networkservices
* delete_networkservice
* get_networkservice_info

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
