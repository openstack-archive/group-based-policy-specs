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
                                                         |  Network Function|
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
    | Network Function |   RPC    | |Configurator|--------->Proxy| Namespace | |
    |      Manager     <------------>   Agent    |  REST | +-----+           | |
    +------------------+          | +-----v------+       +-------------------+ |
             |                    +-------|------------------------------------+
             |                            |
             |Openstack REST              |Configuration Event RPC
             |(nova, neutron, heat)       |
             v                            |


NFP framework implementation will include the following components:

1. A reference Network Function Controller implementation that implements the
   life-cycle management of Network Service VMs and renders the configuration
   of network services deployed on these VMs. The Network Function Controller
   consists of 3 components, the Network Function Manager, the Network Function
   Configurator and a Configurator Proxy.
2. The Network Function Manager runs in the OpenStack management domain and
   provides the orchestration functionality for network services. The Network
   Function Manager manages all the transitions required to render the network
   service. These stages include creation of the network device, creation of
   network interfaces on the device, initial configuration of the network device,
   configuration of the network service and monitoring the network service.
3. The Network Function Configurator is a stateless component that facilitates
   rendering of the configuration required by the Network Function Manager onto
   the network device. For any stage that involves provisioning on the network
   device, the Network Function Manager communicates to the Network Function
   Configurator. The Network Function Configurator runs in the tenant domain.
4. The Configurator Proxy enables the communication between the Network Function
   Manager in the management domain and the Network Function Configurator running
   in the tenant domain. The Configurator Proxy runs on the network node.
5. NFP Node Driver which is a Node Composition Plugin driver to provision Service
   nodes and render the configuration of Service nodes using the Network Function
   Manager.

NFP framework implementation mainly involves APIs for the management of Network
Service (network_function) and Network Service Device (network_function_device)
resources. Network Service is the logical representation that encapsulates all
the instances required to render a network service. Network Service Device
(network_function_device) represents the device (for e.g. a VM) that is used to
run the network service. The APIs defined below use network_function and
network_function_device when referring to a Network Service and Network Service
Device respectively.

Network Function Manager
------------------------

The Network Function Manager listens to RPC messages from NFP Node Driver and
provisions the requested Network Service. The following RPC messages from the
NFP Node Driver are processed:

* create_network_function
* update_network_function
* delete_network_function
* get_network_functions
* get_network_function
* policy_target_added_notification
* policy_target_removed_notification
* consumer_ptg_added_notification
* consumer_ptg_removed_notification
* chain_parameters_updated_notification

The Network Function Manager processes the following notifications received from
the Network Function Configurator via the Configurator Proxy.

* network_function_device_notification

::

 notification_data {
     'resource': <healthmonitor/routes/interfaces>,
     'kwargs': <notify method arguments>
 }

Notifications from the Configurator are received by making a periodic REST call
from the Network Function Manager to the Configurator to check for pending
notifications.

The Network Function Manager implements a pluggable driver framework to provide
life cycle management functionality. This allows for alternate implementations.
A life cycle management driver is required to provide the following methods:

* create_network_function
* delete_network_function_device
* select_network_function_device
* get_network_function_device_status

* plug_network_function_device_interface
* unplug_network_function_device_interface

* get_network_function_device_sharing_info
* get_network_function_device_healthcheck_info
* get_network_function_device_config_info

* get_network_function_config_info

Network Function Configurator
-----------------------------

The Network Function Configurator runs as a VM in the service tenant and exposes
a RESTful API. The Network Function Configurator is stateless and provides the
channel for the Network Function Manager to reach the network services. The
Network Function Configurator implements the following REST APIs:

* create_network_function_device_config
* delete_network_function_device_config

::

 request_data {
     info {
         version: <v1/v2/v3>
     }
     config [
         {
             'resource': <healthmonitor/routes/interfaces>,
             'kwargs': <resource parameters>
         },
         {
             'resource': <healthmonitor/routes/interfaces>,
             'kwargs': <resource parameters>
         }, ...
     ]
 }

* create_network_function_config
* delete_network_function_config

::

  request_data {
     info {
         version: <v1/v2/v3>
         type: <firewall/vpn/loadbalancer>
     }
     config [
         {
             'resource': <resource name>,
             'kwargs': <resource parameters>
         },
         {
             'resource': <resource name>,
             'kwargs': <resource parameters>
         }, ...
     ]
  }

* get_notifications

::

  notifications_data [
     {
         'receiver': <neutron/orchestrator>,
         'resource': <firewall/vpn/loadbalancer/healthmonitor/routes/interfaces>,
         'method': <notification method name>,
         'kwargs': <notification method arguments>
     },
     {
         'receiver': <neutron/orchestrator>,
         'resource': <firewall/vpn/loadbalancer/healthmonitor/routes/interfaces>,
         'method': <notification method name>,
         'kwargs': <notification method arguments>
     }, ...
  ]

In addition to the create_network_function_config REST API, the configurator
also implements REST APIs to consume the Neutron service configuration APIs
as a transition step.

The Network Function Configurator implements a pluggable driver framework to
enable vendor device drivers to be used with the Configurator to configure
vendor devices.

Network Function Configurator Proxy
-----------------------------------

The Configurator Proxy is implemented on the network node as a combination of
Configurator Agent on the network node and a Proxy running in the router
namespace of the service tenant. The Configurator Proxy is required to provide
the communication between the Network Function Manager and the Network Function
Configurator. The Configurator Agent receives RPC messages from the Network
Function Manager and invokes REST APIs over a unix domain socket to the Proxy
in the namespace. The Proxy forwards the REST calls to the Configurator over the
service management network provisioned in the service tenant. In addition to the
RPCs from the Network Function Manager the Configurator Agent also listens and
process RPC messages from Neutron Service plugin drivers as a transition step.

The following RPC messages from the Network Function Manager are processed by the
Configuration Proxy:

* create_network_function_device_config
* delete_network_function_device_config

::

  request_data {
     info {
         version: <v1/v2/v3>
     }
     config [
         {
             'resource': <healthmonitor/routes/interfaces>,
             'kwargs': <resource parameters>
         },
         {
             'resource': <healthmonitor/routes/interfaces>,
             'kwargs': <resource parameters>
         }, ...
     ]
  }

* create_network_function_config
* delete_network_function_config

::

  request_data {
     info {
         version: <v1/v2/v3>
         service_type: <firewall/vpn/loadbalancer>
     }
     config [
         {
             'resource': <resource name>,
             'kwargs': <resource parameters>
         },
         {
             'resource': <resource name>,
             'kwargs': <resource parameters>
         }, ...
     ]
  }

Process Model
-------------

The Network Function Manager and Network Function Congfigurator are implemented
using the python multiprocessing module as a main listener process and a
configurable number of worker processes. The RPC callback running in the context
of the listener process generates an event onto one of the event queues. Each
worker process is assigned to an event queue and handles the events in the queue
by invoking the code required to process the event.

The process model for the Network Function Manager and the Network Function
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


The code in the Network Function Manager and the Network Function Configurator is
organized as modules and drivers. Each module registers RPC handlers and event
handlers. The Network Function Manager includes the Life Cycle Management module.
The Network Function Configurator includes different configuration modules for
LB, FW, VPN service types. The Network Function Configurator also includes a
module to handle events common across all service types. The Network Function
Manager and Network Function Configurator provide a driver framework to
customize the implementation based on the actual device being instantiated to
run the network service.

Data model impact
-----------------

The following resources will be used for the implementation:

1. NetworkFunction

NetworkFunction defines the instantiation of a ServiceChainNode. Creating a
NetworkFunction will instantiate 1 or more instances of the logical service based
on the ServiceProfile. NetworkFunction is the folder of all the instances of the
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

2. NetworkFunctionInstance

NetworkFunctionInstance defines each of the instances of a NetworkFunction.

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
|network_function_id|UUID    |RW, all  |required  |foreign-key  |NetworkFunction|
|                   |        |         |          |             |Id             |
+-------------------+--------+---------+----------+-------------+---------------+
|port_info          |list    |RO, all  |          |foreign-key  |PortInfo ids   |
|                   |(UUID)  |         |          |             |               |
|                   |        |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|ha_state           |string  |RW, all  |''        |             |active or      |
|                   |        |         |          |             |standby HA mode|
+-------------------+--------+---------+----------+-------------+---------------+
|network_function_de|UUID    |RW, all  |required  |foreign-key  |Id of device   |
|vice_id            |        |         |          |             |deploying the  |
|                   |        |         |          |             |FunctionInstanc|
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
|                   |        |         |          |             |gbp_policy_targ|
|                   |        |         |          |             |et             |
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

5. NetworkFunctionDevice

NetworkFunctionDevice defines the device (for e.g. a VM) rendering
NetworkFunctionInstance(s) and the attributes associated with the
NetworkFunctionDevice to manage the network services. A single
NetworkFunctionDevice can render multiple NetworkFunctionInstances(s),
for e.g, a single VM rendering instances of different NetworkFunctions of a
tenant.

+-------------------+--------+---------+----------+-------------+---------------+
|Attribute          |Type    |Access   |Default   |Validation/  |Description    |
|Name               |        |         |Value     |Conversion   |               |
+===================+========+=========+==========+=============+===============+
|id                 |string  |RO, all  |generated |N/A          |identity       |
|                   |(UUID)  |         |          |             |               |
+-------------------+--------+---------+----------+-------------+---------------+
|name               |string  |RW, all  |''        |string       |human-readable |
|                   |        | 	       |          |             |name           |
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
|monitoring_port_net|UUID    |RW, all  |          |foreign-key  |NetworkInfo id |
|work  	            |        |         |          |             |               |
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
