..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Network Function Plugin Framework
=================================


Problem description
===================

GBP provides the mechanism to define a chain of network services [1] to process
the traffic between a provider and multiple consumer Policy Target Groups. The
network services include services such as Firewalls, Load Balancers, VPN. The
network services could be inserted in different modes as addressed by the
Traffic Stitching Plumber specification [2][3].

GBP enables tenant users to define required network services using the
ServiceChainSpec abstraction. In order to render policies using SeviceChainSpec,
GBP requires an engine to create, license, configure, terminate and manage the
services defined by the spec. Management of the network services also requires a
framework to provide an operator view of the deployed network services.

Proposed change
===============

The proposal of the BP is to implement Network Function Plugin Framework (NFP)
in GBP project to handle lifecycle management of network services that includes
creation, deployment, management and resource pooling, monitoring capabilities
of network services. The proposal extends the GBP platform to provide these
capabilities that are common to all network services and is agnostic to the
type and implementation of the network service.

The first version of the implementation will support the following:
1. Creation of Network Services
2. Termination of Network Services
3. Configuration of Network Services
4. Monitoring of Network Services

Functionality such as support for HA, pooling of service instances belonging
to a logical service and scaleup/scaledown of services will be implemented in
the next version.

The Network Function Plugin Framework will adopt an extensible and pluggable
driver based approach. The framework will define an API and resources that the
GBP Node Composition Plugin’s node drivers can make use of. It will also allow
GBP users to correlate logical Service Nodes with the actual service instances
that render them in a particular chain.

The current proposal is to implement a scalable RPC server to listen to
lifecycle and configuration events emitted by the node drivers and render the
configuration on the services being deployed or already deployed.

The components of NFP framework and the interactions are depicted in the diagram
below:

asciiflow::


                                                         +------------------+
                                                         |  Network Service |
                                                         |    Configurator  |
                                                         +-----v------------+
                                                               |
     Tenant                                                    |
  -------------------------------------------------------------|----------------
     Infra (Management)                                        |
                                                               |
   +-------------------+                                       |
   | Node Comp Plugin  |                                       | REST
   +-------------------+                                       |
       +---------------+                                       |
       |NFP Node Driver|          +----------------------------|---------------+
       +------v--------+          |                            |               |
              |                   |   Network Node             |               |
              | RPC               |                            |               |
              |                   |                      +-----|-------------+ |
    +---------v--------+          | +------------+       | +-----+           | |
    |  Network Service |   RPC    | |Configurator|--------->Proxy| Namespace | |
    |      Manager     <------------>   Agent    |  REST | +-----+           | |
    +------------------+          | +-----v------+       +-------------------+ |
             |                    +-------|------------------------------------+
             |                            |
             |Openstack REST              |Configuration Event RPC
             |(nova, neutron, heat)       |
             v                            |


NFP framework implementation will include the following components:

1. A reference Network Service Controller implementation that implements the
   life-cycle management of Network Service VMs and renders the configuration
   of network services deployed on these VMs. The Network Service Controller
   consists of 3 components, the Network Service Manager, the Network Service
   Configurator and a Configurator Proxy.
2. The Network Service Manager runs in the OpenStack management domain and
   provides the orchestration functionality for network services. The Network
   Service Manager manages all the transitions required to render the network
   service. These stages include creation of the network device, creation of
   network interfaces on the device, initial configuration of the network device,
   configuration of the network service and monitoring the network service.
3. The Network Service Configurator is a stateless component that facilitates the
   rendering of the configuration required by the Network Service Manager onto
   the network device. For any stage that involves provisioning on the network
   device, the Network Service Manager communicates to the Network Service
   Configurator. The Network Service Configurator runs in the tenant domain.
4. The Configurator Proxy enables the communication between the Network Service
   Manager in the management domain and the Network Service Configurator running
   in the tenant domain. The Configurator Proxy runs on the network node.
5. NFP Node Driver which is a Node Composition Plugin driver to provision Service
   nodes and render the configuration of Service nodes using the Network Service
   Manager.

NFP framework implementation mainly involves APIs for the mangagement of Network
Service (network_service) and Network Service Device (nsd) resources. Network
Service is the logical representation that enacpsulates all the instances
required to render a network service. Network Serivce Device (nsd) represents
the device (for e.g. a VM) that is used to run the network service. The APIs
defined below use network_service and nsd when referring to a Network Service
and Network Service Device respectively.

Network Service Manager
-----------------------

The Network Service Manager listens to RPC messages from NFP Node Driver and
provisions the requested Network Service. The following RPC messages from the
NFP Node Driver are processed:

 * create_network_service
 * update_network_service
 * delete_network_service
 * get_network_services
 * get_network_service
 * policy_target_added_notification
 * policy_target_removed_notification
 * consumer_ptg_added_notification
 * consumer_ptg_removed_notification
 * chain_parameters_updated_notification

The Network Service Manager processes the following notifications received from
the Network Service Configurator via the Configurator Proxy.

 * nsd_config_notification
 * nsd_healthmonitor_config_notification
 * nsd_health_status_notification

Notifications from the Configurator are received by making a periodic REST call
from the Network Service Manager to the Configurator to check for pending
notifications.

The Network Service Manager implements a pluggable driver framework to provide
life cycle management functionality. This allows for alternate implementations.
A life cycle management driver is required to provide the following methods:

 * create_nsd
 * delete_nsd
 * select_nsd
 * get_nsd_status

 * plug_nsd_interface
 * unplug_nsd_interface

 * get_nsd_sharing_info
 * get_nsd_healthcheck_info
 * get_nsd_config_info

 * get_network_service_config_info

Network Service Configurator
----------------------------

The Network Service Configurator runs as a VM in the service tenant and exposes
a RESTful API. The Network Service Configurator is stateless and provides the
channel for the Network Service Manager to reach the network services. The
Network Service Configurator implements the following REST APIs:

 * create_nsd_config
 * delete_nsd_config

 * create_network_service_config
 * delete_network_service_config

 * create_network_service_healthmonitor
 * delete_network_service_healthmonitor

In addition to the create_service REST API, the configurator allows implements
REST APIs to consume the Neutron service configuration APIs as a transition step.

The Network Service Configurator implements a pluggable driver framework to
enable vendor device drivers to be used with the Configurator to configure
vendor devices.

Network Service Configurator Proxy
----------------------------------

The Configurator Proxy is implemented on the network node as a combination of
Configurator Agent on the network node and a Proxy running in the router
namespace of the service tenant. The Configurator Proxy is required to provide
the communication between the Network Service Manager and the Network Service
Configurator. The Configurator Agent receives RPC messages from the Network
Service Manager and invokes REST APIs over a unix domain socket to the Proxy
in the namespace. The Proxy forwards the REST calls to the Configurator over the
service management network provisioned in the service tenant. In addition to the
RPCs from the Network Service Manager the Configurator Agent also listens and
process RPC messages from Neutron Service plugin drivers as a transition step.

The following RPC messages from the Network Service Manager are processed by the
Configuration Proxy:

 * create_nsd_config
 * delete_nsd_config

 * create_network_service_config
 * delete_network_service_config

 * create_network_service_healthmonitor
 * delete_network_service_healthmonitor

Process Model
-------------

The Network Service Manager and Network Service Congfigurator are implemented
using the python multiprocessing module as a main listener process and a
configurable number of worker processes. The RPC callback running in the context
of the listener process generates an event onto one of the event queues. Each
worker process is assigned to an event queue and handles the events in the queue
by invoking the code required to process the event.

The process model for the Network Service Manager and the Network Service
Configurator is as shown below:

asciiflow::


                                    +-----------------+        +----------+
                                    | +-------------+ |        |          |
                                    | | | | | | | | <----------|  Worker  |
                                    | +-------------+ |        +----------+
                                    |                 |
                                    |                 |        +----------+
                                    | +-------------+ |        |          |
             +-------------+        | | | | | | | | <----------|  Worker  |
             |             |        | +-------------+ |        +----------+
  ----------->  Listener   |-------->                 |
      RPC    |             |        |                 |        +----------+
             +-------------+        | +-------------+ |        |          |
                                    | | | | | | | | <----------|  Worker  |
                                    | +-------------+ |        +----------+
                                    |                 |
                                    |                 |        +----------+
                                    | +-------------+ |        |          |
                                    | | | | | | | | <----------|  Worker  |
                                    | +-------------+ |        +----------+
                                    +-----------------+
                                        Event Queues


The code in the Network Service Manager and the Network Service Configurator is
organized as modules and drivers. Each module registers RPC handlers and event
handlers. The Network Service Manager includes the Life Cycle Management module.
The Network Service Configurator includes different configuration modules for LB,
FW, VPN service types. The Network Service Configurator also includes a module
to handle events common across all service types. The Network Service Manager
and Network Service Configurator provide a driver framework to customize the
implementation based on the actual device being instantiated to run the network
service.

Data model impact
-----------------

The following resources will be used for the implementation:

1. NetworkService

NetworkService defines the instantiation of a ServiceChainNode. Creating a
NetworkService will instantiate 1 or more instances of the logical service based
on the ServiceProfile. NetworkService is the folder of all the instances of the
logical service, for e.g. the active and passive instances of a HA pair.

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
|description        |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |description    |
+-------------------+--------+---------+----------+-------------+---------------+
|tenant_id          |UUID    |RW, all  |''        |             |tenant id      |
|                   |        |         |          |             |               |
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
|heat_stack_id      |UUID    |RO, all  |          |             |               |
|                   |        |         |          |             |               |
|                   |        |         |          |             |               |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+
|status_description |string  |RO, all  |          |             |description    |
+-------------------+--------+---------+----------+-------------+---------------+

2. NetworkServiceInstance

NetworkServiceInstance defines each of the instances of a NetworkService.

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
|tenant_id          |UUID    |RW, all  |''        |             |tenant id      |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|description        |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |description    |
+-------------------+--------+---------+----------+-------------+---------------+
|network_service_id |UUID    |RW, all  |required  |foreign-key  |NetworkService |
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|port_info          |list    |RO, all  |          |foreign-key  |PortInfo ids   |
|                   |(UUID)  |         |          |             |               |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|ha_state           |string  |RW, all  |''        |             |active or      |
|                   |        |         |          |             |standby HA mode|
+-------------------+--------+---------+----------+-------------+---------------+
|nsd_id             |UUID    |RW, all  |required  |foreign-key  |Id of device   |
|                   |        |         |          |             |deploying the  |
|                   |        |         |          |             |ServiceInstance|
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+

3. PortInfo

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|port_policy        |string  |RW, all  |''        |string       |neutron_port or|
|                   |        |         |          |             |gbp_policy_    |
|                   |        |         |          |             |target         |
+-------------------+--------+---------+----------+-------------+---------------+
|port_classification|string  |RW, all  |''        |             |provider or    |
|                   |        |         |          |             |consumer       |
+-------------------+--------+---------+----------+-------------+---------------+
|port_type          |string  |RW, all  |''        |string       |active, standby|
|                   |        |         |          |             |or master      |
+-------------------+--------+---------+----------+-------------+---------------+

4. NetworkInfo

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|network_policy     |string  |RW, all  |''        |string       |neutron_network|
|                   |        |         |          |             |or gbp_group   |
+-------------------+--------+---------+----------+-------------+---------------+

5. NetworkServiceDevice

NetworkSerivceDevice defines the device (for e.g. a VM) rendering
NetworkServiceInstance(s) and the attributes associated with the NetworkServiceDevice
to manage the netowrk services. A single NetworkServiceDevice can render multiple
NetworkServiceInstances(s), for e.g, a single VM rendering instances of different
NetworkServices of a tenant.

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
|tenant_id          |UUID    |RW, all  |''        |             |tenant id      |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|description        |string  |RW, all  |''        |string       |human-readable |
|                   |        |         |          |             |description    |
+-------------------+--------+---------+----------+-------------+---------------+
|mgmt_port_id       |UUID    |RW, all  |required  |foreign-key  |management     |
|                   |        |         |          |             |PortInfo id    |
+-------------------+--------+---------+----------+-------------+---------------+
|monitoring_port_id |UUID    |RW, all  |          |foreign-key  |PortInfo id    |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|monitoring_port_   |UUID    |RW, all  |          |foreign-key  |NetworkInfo id |
|network            |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|service_vendor     |string  |RO, all  |          |             |vendor         |
+-------------------+--------+---------+----------+-------------+---------------+
|status             |string  |RO, all  |          |             |status         |
+-------------------+--------+---------+----------+-------------+---------------+


REST API impact
---------------


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
