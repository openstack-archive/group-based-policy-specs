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
to a logical service and scaleup/scaledown of services will be defined in a
follow up spec.

The Network Function Plugin Framework will adopt an extensible and pluggable
driver based approach. The framework will define an API and resources that the
GBP Node Composition Plugin’s node drivers can make use of. It will also allow
users to correlate GBP Service Node resources with the actual service instances
that render them in a particular chain.

The current proposal is to implement a scalable RPC server to listen to
lifecycle and configuration events emitted by the node drivers and render the
configuration on the services being deployed or already deployed.

The components of NFP framework and the interactions are depicted in the diagram
below:

asciiflow::


                                                         +------------------+
                                                         | Network Function |
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
    |   Orchestrator   <------------>   Agent    |  REST | +-----+           | |
    +-------------v----+          | +------------+       +-------------------+ |
        |         |               +--------------------------------------------+
        |         |
        |         |Configuration Event RPC
        |         |
        |
        |Openstack REST
        |(nova, neutron heat)
        |
        v


NFP framework implementation will include the following components:

* Network Function Controller: A reference implementation that implements the
  life-cycle management of Network Service VMs and renders the configuration
  of network services deployed on these VMs. The Network Function Controller
  consists of 3 components, the Network Function Orchestrator, the Network
  Function Configurator and a Configurator Proxy.
* Network Function Orchestrator: runs in the OpenStack management domain and
  provides the orchestration functionality for network services. The Network
  Function Orchestrator manages all the transitions required to render the network
  service. These stages include creation of the network device, creation of
  network interfaces on the device, initial configuration of the network device,
  configuration of the network service and monitoring the network service.
* Network Function Configurator: is a stateless component that facilitates
  rendering of the configuration required by the Network Function Orchestrator
  on the network device. For any step that involves provisioning on the network
  device, the Network Function Orchestrator communicates to the Network Function
  Configurator. The Network Function Configurator runs in the tenant domain. The
  Network Function Configurator is a single instance for all tenants and is
  instantiated when the Openstack environment is brought up.
* Configurator Proxy: enables the communication between the Network Function
  Orchestrator in the management domain and the Network Function Configurator
  running in the tenant domain. The Configurator Proxy runs on the network node.
  The network node is a compute node where tenant routers are deployed and
  provides a bridge between the infra network and tenant networks. Alternately
  any compute node that provides this connectivity can be used to deploy the
  Configurator proxy. The Configurator Proxy is instantiated when the Openstack
  environment is brought up.
* NFP Node Driver: which is a Node Composition Plugin driver to provision Service
  nodes and render the configuration of Service nodes using the Network Function
  Orchestrator.

NFP framework implementation mainly involves APIs for the management of Network
Function (network_function) and Network Function Device (network_function_device)
resources. Network Function is the logical representation that encapsulates all
the instances of a network service, for e.g. a pair of instances for HA. Network
Function Instance represents a single instance of the network service. Network
Function Device (network_function_device) represents the device (for e.g. a VM)
that is used to run the network service. A single Network Function Device can
run multiple Network Function Instances, for e.g. a single VM can run multiple
instances of a Firewall service. The term Network Function maps to Network
Services such as a LoadBalancer, Firewall, IDS.

The relationship of the managed resources is as shown below:

asciiflow::


            +------------------+                    +------------------+
            |                  |                    |                  |
            | Network Function |1                  n| Network Function |
            |                  +-------------------->     Instance     |
            |                  |                    |                  |
            +------------------+                    +---------+--------+
                                                              |n
                                                              |
                                                              |1
                                                    +---------V--------+
                                                    |                  |
                                                    | Network Function |
                                                    |      Device      |
                                                    |                  |
                                                    +------------------+


Network Function Orchestrator
-----------------------------

The Network Function Orchestrator listens to RPC messages from NFP Node Driver
and provisions the requested Network Service. The following RPC messages from
the NFP Node Driver are processed:

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

The Network Function Orchestrator processes the following notifications received
from the Network Function Configurator via the Configurator Proxy.

* network_function_notification

Notifications from the Configurator are received by making a periodic REST call
from the Configurator Proxy to the Configurator to check for pending
notifications. Notifications are generated by the Configurator asynchonoulsy,
but need to pulled periodically by the Configurator Proxy as the Configurator
running in the tenant domain doesn't have a mechanism to deliver the
notifications to the infra components.

The Network Function Orchestrator implements a pluggable driver framework to
provide life cycle management functionality. This allows for alternate
implementations. A life cycle management driver is required to provide the
following methods:

* create_network_function_device(device_data)

  Create a Network Function Device. In case there is no hotplug support the
  device is created with the specified ports.

::

  Parameters:
    device_data
    {
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'name': <str>,                  # name to be used while creating the device instance
        'management_network_info': {
            'id': <UUID>                # Management GBP PTG or Neutron Network UUID
        },
        'ports': [                      # Device ports for provider and stitching PTGs
            {'id': <UUID>,              # Provider PT or neutron port id
             'port_classification': <str>, # provider, consumer
             'port_model': <str>},      # Model: gbp, neutron
            {'id': <UUID>,
             'port_classification': <str>,
             'port_model': <str>},
        ],
    }

  Return Value:
    {
        'id': <UUID>,                   # device instance UUID
        'name': <str>,                  # name
        'mgmt_ip_address': <IPv4>,      # management IP address
        'mgmt_port_id': {
            'id': <UUID>,               # PT or neutron port id
            'port_classification': <str>, # management
            'port_model': <str>         # Model: gbp, neutron
        },
        'max_interfaces': <int>,        # maximum interfaces supported by the device
        'interfaces_in_use': <int>,     # number of interfaces the device instance is created with
        'description': <str>            # Description
    }

    None                                # On failure

* delete_network_function_device(device_data)

  Delete a Network Function Device, if 'id' is specified in parameters
  Delete PTs or neutron ports, if 'id' is not specified in parameters

::

  Parameters:
    device_data
    {
        'id': <UUID>,                   # network function device id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'mgmt_port_id': {
            'id': <UUID>,               # PT or neutron port id
            'port_classification': <str>, # management
            'port_model': <str>         # Model: gbp, neutron
        },
    }

  Return Value:
      None

* select_network_function_device(devices, device_data)

  Select a device to use from a list of available devices and ensure that the
  device has sufficient interfaces for the specified ports

::

  Parameters:
    devices
    [
        {'id': <UUID>,                  # device instance UUID
         'name': <str>,                 # name
         'mgmt_ip_address': <IPv4>,     # management IP address
         'mgmt_port_id': {
             'id': <UUID>,              # PT or neutron port id
             'port_classification': <str>, # management
             'port_model': <str>        # Mode: gbp, neutron
         },
         'max_interfaces': <int>,       # maximum interfaces supported by the device
         'interfaces_in_use': <int>,    # number of interfaces the device instance is created with
         'description': <str>           # Description
        },
        {'id': <UUID>,                  # device instance UUID
         'name': <str>,                 # name
         ...
        }
    ]

    device_data
    {
        'ports': [                      # Device ports for provider and stitching PTGs
            {'id': <UUID>,              # Provider PT or neutron port id
             'port_classification': <str>, # provider, consumer
             'port_model': <str>},      # Mode: gbp, neutron
            {'id': <UUID>,
             'port_classification': <str>,
             'port_model': <str>},
        ],
    }

  Return Value:
    {
        'id': <UUID>,                   # device instance UUID
        'name': <str>,                  # name
        'mgmt_ip_address': <IPv4>,      # management IP address
        'mgmt_port_id': {
            'id': <UUID>,               # PT or neutron port id
            'port_classification': <str>, # management
            'port_model': <str>         # Mode: gbp, neutron
        },
        'max_interfaces': <int>,        # maximum interfaces supported by the device
        'interfaces_in_use': <int>,     # number of interfaces the device instance is created with
        'description': <str>            # Description
    }

    None                                # On failure

* get_network_function_device_status(device_data)

  Get the status of the network function device

::

  Parameters:
    device_data
    {
        'id': <UUID>,                   # network function device id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
    }

  Return Value:
      None      # Failure
      <str>     # status string

* plug_network_function_device_interface(device_data)

  Attach the specified ports to the hotplug capable network function device

::

  Parameters:
    device_data
    {
        'id': <UUID>,                   # network function device id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'ports': [                      # Device ports for provider and stitching PTGs
            {'id': <UUID>,              # Provider PT or neutron port id
             'port_classification': <str>, # provider, consumer
             'port_model': <str>},      # Mode: gbp, neutron
            {'id': <UUID>,
             'port_classification': <str>,
             'port_model': <str>},
        ],
    }

  Return Value:
    <bool>      # True on success, False on failure

* unplug_network_function_device_interface(device_data)

  Detach the ports from the hotplug capable network function device

::

  Parameters:
    device_data
    {
        'id': <UUID>,                   # network function device id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'ports': [                      # Device ports for provider and stitching PTGs
            {'id': <UUID>,              # Provider PT or neutron port id
             'port_classification': <str>, # provider, consumer
             'port_model': <str>},      # Mode: gbp, neutron
            {'id': <UUID>,
             'port_classification': <str>,
             'port_model': <str>},
        ],
    }

  Return Value:
      <bool>    # True on success, False on failure

* get_network_function_device_sharing_info(device_data)

  Get the filters to use in building the list of network function devices
  that can be shared

::

  Parameters:
    device_data
    {
        'tenant_id': <UUID>,            # tenant id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
    }

  Return Value:
    {
        'filters': {
            'key': 'value',
            ...
        }
    }

* get_network_function_device_healthcheck_info(device_data)

  Get the health check information to be configured on the netowrk
  function device

::

  Parameters:
    device_data
    {
        'id': <UUID>,                   # network function id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'mgmt_ip_address': <IPv4>,      # management IP address
    }

  Return Value:
    {
        'config': [
            {
                'resource': 'healthmonitor',
                'resource_data': {
                    <key>: <value>
                    ...
                }
            }
        ]
    }

* get_network_function_device_config_info(device_data)

  Get the device configuration parameters to be configured on the network
  function device

::

  Parameters:
    device_data
    {
        'tenant_id': <UUID>,            # tenant id
        'id': <UUID>,                   # network function id
        'service_details': {
            'service_vendor': <str>,    # Service Vendor: vyos, haproxy
            'service_type': <str>,      # Service Type: firewall, lb, vpn
            'device_type': <str>,       # Device Type: None, VM
            'network_mode': <str>       # Mode: gbp, neutron
        },
        'mgmt_ip_address': <IPv4>,      # management IP address
        'ports': [                      # Device ports for provider and stitching PTGs
            {'id': <UUID>,              # Provider PT or neutron port id
             'port_classification': <str>, # provider, consumer
             'port_model': <str>},      # Mode: gbp, neutron
            {'id': <UUID>,
             'port_classification': <str>,
             'port_model': <str>},
        ],
    }

  Return Value:
    {
        'config': [
            {
                'resource': 'interfaces',
                'resource_data': {
                    <key>: <value>
                    ...
                }
            },
            {
                'resource': 'routes',
                'resource_data': {
                    <key>: <value>
                    ...
                }
            },
        ]
    }

Network Function Configurator
-----------------------------

The Network Function Configurator runs as a VM in the service tenant and exposes
a RESTful API. The Network Function Configurator is stateless and provides the
channel for the Network Function Orchestrator to reach the network services. The
Network Function Configurator implements the following REST APIs:

* create_network_function_device_config

 POST /v1/nfp/create_network_function_device_config

 Response code:          200
 Error code:             400
 Request parameters

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|info                 |dict        |Contains the header info       |
|                     |            |common to all elements in the  |
|                     |            |config list                    |
+---------------------+------------+-------------------------------+
|config               |list        |list of network function       |
|                     |            |device config elements         |
+---------------------+------------+-------------------------------+

The info dict takes the following parameters as keys. All parameters
are required.

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|service_vendor       |string      |Vendor name                    |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|service_type         |string      |Type of service. Values are    |
|                     |            |firewall, lb, vpn              |
+---------------------+------------+-------------------------------+
|context              |dict        |opaque_data that is returned   |
|                     |            |with async notifications       |
|                     |            |posted in response to the      |
|                     |            |request                        |
+---------------------+------------+-------------------------------+

The config list comprises of one or more dicts, each of which has the
following parameters as keys. In every dict both parameters are required.

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|resource             |string      |Identifies the resource for    |
|                     |            |the configuration. Values are  |
|                     |            |healthmonitor, interfaces,     |
|                     |            |routes                         |
+---------------------+------------+-------------------------------+
|resource_data        |dict        |Specifies the attributes for   |
|                     |            |each resource to be configured |
+---------------------+------------+-------------------------------+

It is always required to specify a healthmonitor resource. The resource_data
for the healthmonitor has the following parameters. The parameters serve to
monitor the health of the service VM.

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|vmid                 |uuid        |Id of network function device  |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|mgmt_ip              |string      |management port IP address     |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|periodicity          |integer     |healthmonitor poll interval    |
|                     |            |                               |
+---------------------+------------+-------------------------------+

An interface resource can be optionally specified. The resource_data
for the interface has the following parameters. These parameters are used to
configure the interfaces in the service VM.

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
+---------------------+------------+-------------------------------+
|mgmt_ip              |string      |management port IP address     |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|provider_ip          |string      |provider port IP address       |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|provider_cidr        |string      |provider port CIDR             |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|provider_interface_i |integer     |index in device interface      |
|ndex                 |            |table (optional)               |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|provider_mac         |string      |provider port mac address      |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|stitching_ip         |string      |stitching port IP address      |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|stitching_cidr       |string      |stitching port CIDR            |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|stitching_interface_i|            |index in device interface      |
|ndex                 |            |table (optional)               |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|stitching_mac        |string      |stitching port mac address     |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|service_id           |uuid        |Id of network function         |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|tenant_id            |uuid        |Id of owning tenant            |
|                     |            |                               |
+---------------------+------------+-------------------------------+

A routes resource can be optionally specified. The resource_data
for the routes resource can be used to configure routes and Policy Based
Routing rules on the service VM.

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|mgmt_ip              |IP address  |management port IP address     |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|source_cidrs         |list        |list of source CIDRs for PBR   |
|                     |            |config                         |
+---------------------+------------+-------------------------------+
|destination_cidr     |string      |route destination CIDR         |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|gateway_ip           |IP address  |route nexthop address          |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|provider_interface_i |integer     |index in device interface      |
|ndex                 |            |table (optional)               |
|                     |            |                               |
+---------------------+------------+-------------------------------+

Response parameters
None

* delete_network_function_device_config

POST /v1/nfp/delete_network_function_device_config

Response code:          200
Error code:             400
Request parameters
The request parameters are identical to create_network_function_device_config

Response parameters
None

Example request body:

::

 {
     'info': {
         'service_vendor': 'vyos',
         'service_type': 'firewall',
         'context': {
             'nf_id': 'a87cc70a-3e15-4acf-8205-9b711a3531b7',
             'nfi_id': 'a87cc70a-3e15-4acf-8205-9b711a3531b7',
             'nfd_id': 'a87cc70a-3e15-4acf-8205-9b711a3531b7',
             'nfd_ip': '10.0.1.30',
             'operation': 'create',
         }
     },
     'config': [
         {
             'resource': 'healthmonitor',
             'resource_data': {
                 'vmid': 'a87cc70a-3e15-4acf-8205-9b711a3531b7',
                 'mgmt_ip': '10.0.1.20',
                 'periodicity': 10,
             }
         },
         {
             'resource': 'interfaces',
             'resource_data': {
                 ...
                 ...
             }
         },
     ]
 }

* create_network_function_config

 POST /v1/nfp/create_network_function_config

 Response code:          200
 Error code:             400
 Request parameters

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|info                 |dict        |Contains the header info       |
|                     |            |common to all elements in the  |
|                     |            |config list                    |
+---------------------+------------+-------------------------------+
|service_vendor       |string      |Vendor name                    |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|service_type         |string      |Type of service. Values are    |
|                     |            |nfp_service, firewall, lb, vpn |
+---------------------+------------+-------------------------------+
|config               |list        |list of network function       |
|                     |            |config elements                |
+---------------------+------------+-------------------------------+
|context              |dict        |opaque_data that is returned   |
|                     |            |with async notifications       |
|                     |            |posted in response to the      |
|                     |            |request                        |
+---------------------+------------+-------------------------------+
|resource             |string      |Identifies the resource for    |
|                     |            |the configuration. Values are  |
|                     |            |heat, ansible, nas_res_type.   |
|                     |            |nas_res_type specifies neutron |
|                     |            |advanced service resources     |
|                     |            |such as vip, pool, firewall.   |
+---------------------+------------+-------------------------------+
|resource_data        |dict        |In case resource is heat or    |
|                     |            |ansible, resource_data         |
|                     |            |contains key-value pair for    |
|                     |            |config_string below.           |
|                     |            |In case resource is            |
|                     |            |nas_res_type,                  |
|                     |            |contains implementation        |
|                     |            |dependent key-value pairs.     |
+---------------------+------------+-------------------------------+
|config_string        |string      |Speficies a heat template,     |
|                     |            |ansible config.                |
+---------------------+------------+-------------------------------+

Response parameters
None

* delete_network_function_config

POST /v1/nfp/delete_network_function_config

Response code:          200
Error code:             400
Request parameters
The request parameters are identical to create_network_function_config

Response parameters
None

Example request body:

::

 {
     'info': {
         'service_vendor': '',
         'service_type': 'nfp_service',
         'context': {
         }
     }
     'config': [
         {
             'resource': 'heat',
             'resource_data': {
                 'config_string': '',
             }
         },
     ]
 }

 {
     'info': {
         'service_vendor': '',
         'service_type': 'lb',
         'context': {
         }
     }
     'config': [
         {
             'resource': 'vip',
             'resource_data': {
                 'key': 'value',
                 ...
                 ...
             }
         },
     ]
 }

* get_notifications

 GET /v1/nfp/get_notifications

 Response code:          200
 Error code:             400
 Response parameters

+---------------------+------------+-------------------------------+
|Parameter            |Type        |Description                    |
|                     |            |                               |
+=====================+============+===============================+
|info                 |dict        |Contains the header info       |
|                     |            |common to all elements in the  |
|                     |            |notification list              |
+---------------------+------------+-------------------------------+
|service_type         |string      |Type of service. Values are    |
|                     |            |nfp_service, firewall, lb,     |
|                     |            |vpn. This value should be the  |
|                     |            |service_type from the request  |
+---------------------+------------+-------------------------------+
|context              |dict        |opaque_data from the request   |
|                     |            |that resulted in this          |
|                     |            |notification.                  |
+---------------------+------------+-------------------------------+
|notification         |list        |list of network function       |
|                     |            |notification elements          |
+---------------------+------------+-------------------------------+
|resource             |string      |Identifies the resource the    |
|                     |            |notification applies to.       |
|                     |            |Values are healthmonitor,      |
|                     |            |interface, routes, heat,       |
|                     |            |ansible, nas_res_type.         |
|                     |            |nas_res_type specifies neutron |
|                     |            |advanced service resources     |
|                     |            |such as vip, pool, firewall.   |
+---------------------+------------+-------------------------------+
|data                 |dict        |notification data              |
|                     |            |                               |
+---------------------+------------+-------------------------------+
|status_code          |string      |Values are success, failure,   |
|                     |            |unhandled.                     |
+---------------------+------------+-------------------------------+
|error_msg            |string      |error message (optional)       |
|                     |            |                               |
+---------------------+------------+-------------------------------+

Example response body:

::

 {
     'info': {
         'service_type': 'nfp_service',
         'context': {
         }
     },
     'notification': [
         {
             'resource': 'heat',
             'data': {
                 'status_code': 'success',
                 'error_msg': '',
             }
         },
     ]
 }

 {
     'info': {
         'service_type': 'lb',
         'context': {
         }
     },
     'notification': [
         {
             'resource': 'vip',
             'data': {
                 'key': 'value',
                 ...
             }
         },
     ]
 }

The get_notifications API provides the mechanism for the orchestrator to poll
for any notifications from the configurator. The notifications need to be polled
as the configurator running as a service tenant VM doesn't have the capability
to initiate the communication.

The Network Function Configurator implements a pluggable driver framework to
enable vendor device drivers to be used with the Configurator to configure
vendor devices. The framework allows for multiple drivers to be configured for
a network function and selects the driver to use based on the service_flavor
specified in ServiceProfile.

Network Function Configurator Proxy
-----------------------------------

The Configurator Proxy is implemented on the network node as a combination of
Configurator Agent on the network node and a Proxy running in the router
namespace of the service tenant. The Configurator Proxy is required to provide
the communication between the Network Function Orchestrator and the Network
Function Configurator. The Configurator Agent receives RPC messages from the
Network Function Orchestrator and invokes REST APIs over a unix domain socket to
the Proxy in the namespace. The Proxy forwards the REST calls to the Network
Function Configurator over the service management network provisioned in the
service tenant. The 'service' tenant is a project created by default on an
OpenStack install. Service management network is created in this tenant during
installation and used as the management network for any Network Function Device
created for a tenant.

The following RPC messages from the Network Function Orchestrator are received
and proxied to the Network Function Configurator by the Configuration Proxy:

* create_network_function_device_config
* delete_network_function_device_config
* create_network_function_config
* delete_network_function_config

Process Model
-------------

The Network Function Orchestrator and Network Function Congfigurator are
implemented using the python multiprocessing module as a main listener process
and a configurable number of worker processes. The RPC callback running in the
context of the listener process generates an event onto one of the event queues.
Each worker process is assigned to an event queue and handles the events in the
queue by invoking the code required to process the event.

The process model for the Network Function Orchestrator and the Network Function
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


The Listener process runs the RPC handlers for the Network Function Orchestrator
and the Network Function Configurator and implements minimal processing in
these handlers. The bulk of processing is offloaded to the Worker processes
by posting an event into one of the event queues.

The code in the Network Function Orchestrator and the Network Function
Configurator is organized as modules and drivers. Each module registers RPC
handlers and event handlers. The Network Function Orchestrator includes the
Life Cycle Management module. The Network Function Configurator includes
different configuration modules for LB, FW, VPN service types. The Network
Function Configurator also includes a module to handle events common across all
service types. The Network Function Orchestrator and Network Function
Configurator provide a driver framework to customize the implementation based on
the actual device being instantiated to run the network service.

Data model impact
-----------------

The following resources will be used for the implementation. The resources
described in this section are currently used internally by NFP and not exposed
via a tenant API. These resources will be exposed to the user at a later stage
to provide an operational view of the network functions and devices rendering
the GBP service chains. The "Access" column specifics are relevant only when
these resources are exposed in the tenant API.

1. NetworkFunction

NetworkFunction defines the instantiation of a ServiceChainNode. Creating a
NetworkFunction will instantiate 1 or more instances of the logical service based
on the ServiceProfile. NetworkFunction is the folder of all the instances of the
logical service, for e.g. the active and passive instances of a HA pair.

+-------------------+--------+---------+----------+------------+----------------+
|Attribute          |Type    |Access   |Default   |Validation/ |Description     |
|Name               |        |         |Value     |Conversion  |                |
+===================+========+=========+==========+============+================+
|id                 |string  |RO, all  |generated |N/A         |identity        |
|                   |(UUID)  |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|name               |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |name            |
+-------------------+--------+---------+----------+------------+----------------+
|description        |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+
|tenant_id          |UUID    |RW, all  |''        |            |tenant id       |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|service_id         |UUID    |RW, all  |required  |            |GBP Service     |
|                   |        |         |          |            |Node Id or      |
|                   |        |         |          |            |Neutron         |
|                   |        |         |          |            |Service Id      |
+-------------------+--------+---------+----------+------------+----------------+
|service_chain_id   |UUID    |RW, all  |          |            |GBP Service     |
|                   |        |         |          |            |Chain Instance  |
|                   |        |         |          |            |Id              |
+-------------------+--------+---------+----------+------------+----------------+
|service_profile_id |UUID    |RW, all  |          |            |Service Profile |
|                   |        |         |          |            |Id              |
+-------------------+--------+---------+----------+------------+----------------+
|service_config     |string  |RW, all  |          |            |Device Specific |
|                   |        |         |          |            |Configuration   |
+-------------------+--------+---------+----------+------------+----------------+
|heat_stack_id      |UUID    |RO, all  |          |            |                |
|                   |        |         |          |            |                |
|                   |        |         |          |            |                |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|status             |string  |RO, all  |          |            |status          |
+-------------------+--------+---------+----------+------------+----------------+
|status_description |string  |RO, all  |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+

2. NetworkFunctionInstance

NetworkFunctionInstance defines each of the instances of a NetworkFunction.

+-------------------+--------+---------+----------+------------+----------------+
|Attribute          |Type    |Access   |Default   |Validation/ |Description     |
|Name               |        |         |Value     |Conversion  |                |
+===================+========+=========+==========+============+================+
|id                 |string  |RO, all  |generated |N/A         |identity        |
|                   |(UUID)  |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|name               |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |name            |
+-------------------+--------+---------+----------+------------+----------------+
|tenant_id          |UUID    |RW, all  |''        |            |tenant id       |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|description        |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+
|network_function_id|UUID    |RW, all  |required  |foreign-key |NetworkFunction |
|                   |        |         |          |            |Id              |
+-------------------+--------+---------+----------+------------+----------------+
|port_info          |list    |RO, all  |          |foreign-key |PortInfo ids    |
|                   |(UUID)  |         |          |            |                |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|ha_state           |string  |RW, all  |''        |            |active or       |
|                   |        |         |          |            |standby mode    |
+-------------------+--------+---------+----------+------------+----------------+
|network_function_de|UUID    |RW, all  |required  |foreign-key |Id of device    |
|vice_id            |        |         |          |            |deploying the   |
|                   |        |         |          |            |Function        |
|                   |        |         |          |            |Instance        |
+-------------------+--------+---------+----------+------------+----------------+
|status             |string  |RO, all  |          |            |status          |
+-------------------+--------+---------+----------+------------+----------------+
|status_description |string  |RO, all  |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+

3. PortInfo

+-------------------+--------+---------+----------+------------+----------------+
|Attribute          |Type    |Access   |Default   |Validation/ |Description     |
|Name               |        |         |Value     |Conversion  |                |
+===================+========+=========+==========+============+================+
|id                 |string  |RO, all  |generated |N/A         |identity        |
|                   |(UUID)  |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|port_model         |string  |RW, all  |''        |string      |neutron_port or |
|                   |        |         |          |            |gbp_policy_targ |
|                   |        |         |          |            |et              |
+-------------------+--------+---------+----------+------------+----------------+
|port_classification|enum    |RW, all  |''        |            |provider or     |
|                   |        |         |          |            |consumer        |
+-------------------+--------+---------+----------+------------+----------------+
|port_role          |enum    |RW, all  |''        |            |active, standby |
|                   |        |         |          |            |or master       |
+-------------------+--------+---------+----------+------------+----------------+

4. NetworkInfo

+-------------------+--------+---------+----------+------------+----------------+
|Attribute          |Type    |Access   |Default   |Validation/ |Description     |
|Name               |        |         |Value     |Conversion  |                |
+===================+========+=========+==========+============+================+
|id                 |string  |RO, all  |generated |N/A         |identity        |
|                   |(UUID)  |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|network_model      |enum    |RW, all  |''        |            |neutron_network |
|                   |        |         |          |            |or gbp_group    |
+-------------------+--------+---------+----------+------------+----------------+

5. NetworkFunctionDevice

NetworkFunctionDevice defines the device (for e.g. a VM) rendering
NetworkFunctionInstance(s) and the attributes associated with the
NetworkFunctionDevice to manage the network services. A single
NetworkFunctionDevice can render multiple NetworkFunctionInstances(s),
for e.g, a single VM rendering instances of different NetworkFunctions of
a tenant.

+-------------------+--------+---------+----------+------------+----------------+
|Attribute          |Type    |Access   |Default   |Validation/ |Description     |
|Name               |        |         |Value     |Conversion  |                |
+===================+========+=========+==========+============+================+
|id                 |string  |RO, all  |generated |N/A         |identity        |
|                   |(UUID)  |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|name               |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |name            |
+-------------------+--------+---------+----------+------------+----------------+
|tenant_id          |UUID    |RW, all  |''        |            |tenant id       |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|description        |string  |RW, all  |''        |string      |human-readable  |
|                   |        |         |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+
|mgmt_ip_address    |String  |RW, all  |required  |String      |management      |
|                   |        |         |          |            | IP Address     |
+-------------------+--------+---------+----------+------------+----------------+
|mgmt_port_id       |UUID    |RW, all  |required  |foreign-key |management      |
|                   |        |         |          |            |PortInfo id     |
+-------------------+--------+---------+----------+------------+----------------+
|monitoring_port_id |UUID    |RW, all  |          |foreign-key |PortInfo id     |
|                   |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|monitoring_port_net|UUID    |RW, all  |          |foreign-key |NetworkInfo id  |
|work               |        |         |          |            |                |
+-------------------+--------+---------+----------+------------+----------------+
|service_vendor     |string  |RO, all  |          |            |vendor          |
+-------------------+--------+---------+----------+------------+----------------+
|status             |string  |RO, all  |          |            |status          |
+-------------------+--------+---------+----------+------------+----------------+
|status_description |string  |RO, all  |          |            |description     |
+-------------------+--------+---------+----------+------------+----------------+


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
