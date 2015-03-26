..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================
Traffic Steering as a Service Chaining provider
===============================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/traffic-steering-sfc

The Service Chaining API provided by the Group-based Policy (GBP) project
currently allows the instantiation of chains of network services based on the
deployment requirements of applications. However, it is currently only possible
to use either Firewall or Load Balancer services based on Heat orchestration
templates, and chain them together. This blueprint proposes a set of
modifications to allow a new flavor of service functions to be deployed using
Group-based Policy. This flavor is called Traffic Steering (TS) and is
particularly helpful in chaining services that are not known to Neutron and
only manifested as Neutron ports (e.g. a user managed service VM).
Furthermore, these virtual machines can be used as service functions, giving
cloud tenants or NFV operators the freedom to choose their own network services
based on VMs or anything else that can be mapped to Neutron ports,
and chain them together. This spec builds upon the spec presented in [1]_ and
its underlying implementation, but now adapted and aimed at Group-based Policy.


Problem description
===================

OpenStack Neutron provides users the ability to instantiate and manage network
services in an easy and convenient way through well-defined REST APIs.
Resources are pooled and allocated to tenants with no human interaction besides
users themselves requesting the service.

The momentum around the Network Functions Virtualization (NFV) concept,
primarily lead by a dedicated ETSI Industry Specification Group (ISG) [2]_, is
fostering this latter fact. According to ETSI, OpenStack is well positioned to
be the reference platform to play the Virtual Infrastructure Manager (VIM) role
in their reference architecture [3]_ [4]_. However, there are still gaps that
need to be covered in order to make this possible. We believe that this
blueprint identifies and tries to fill part of this gap.

The term Service Function Chain (SFC) is today widely referred within the NFV
context (also referred by Virtual Network Function (VNF) Forwarding Graph
within the ETSI NFV ISG). A requirement in the scope of SFC is the need to
steer traffic at the granularity of subscriber and traffic types. This is in
fact in line with how [5]_ has (loosely) defined an SFC - "an ordered set of
service functions that must be applied to packets and/or frames selected as a
result of classification".

NFV is not the only scenario where Service Chaining can be useful. Cloud based
datacenters can also benefit from the chaining of different services, like
load balancers, firewalls or VPNs.

Earlier, the Neutron Advanced Services team orchestrated and proposed a
comprehensive Advanced Network Services Common Framework [7]_ as an enhanced
proposal from [8]_. The framework was an umbrella for, among others, the
blueprint that preceded this one [1]_ and the
Service Chaining specification [9]_.

The Service Chaining API provided by the Group-based Policy (GBP) project
already allows the instantiation of chains of network services based on the
deployment requirements of applications. However, it is currently only possible
to use either Firewall or Load Balancer services based on Heat orchestration
templates, and chain them together. This blueprint aims at providing a new
abstraction to allow a new flavor of service functions to be used on service
chains by recurring to Traffic Steering between generic Neutron ports.

There are several network service types OpenStack could contemplate in the
future as Neutron network services (e.g. deep packet inspection (DPI),
intrusion detection and prevention systems (IDS/IPS)), but several others that
it will probably never contemplate (e.g. Evolved Packet Core (EPC), IP
Multimedia Subsystem (IMS)). Either way, OpenStack should provide a strong
foundation for the deployment of network services, whether natively or not. In
other words, OpenStack can in fact have embedded network services (Neutron
network services) as it already has and continue to explore them, but it should
also provide the foundations for network services to be deployed within
OpenStack without the platform knowing the service itself. The proposal in
[6]_ already tackled part of this.

Given the limited collection of network services types and providers Neutron
provides, it is currently not possible to deploy a network service software of
any type or provider in a machine (virtual or bare metal machine) instance that
Neutron does not support, and hook it at strategically placed points on the
network. An example is the deployment of a DPI instance running on a machine
connected to a Neutron network and place it in between an edge router and a
private network where tenant’s machines are connected. Inbound/outbound traffic
should be intercepted and steered to the machine running the DPI instance first
and then forwarding it to the original destination. The same scenario can be
applied to a firewall, IDS and IPS.


Proposed change
===============

Every Neutron network services and machines are connected to a Neutron network
via Neutron ports. A Traffic Steering model that covers both network services
and machines should be possible. Neutron network services and machines should
have no notion of Traffic Steering. Rather a Traffic Steering model only knows
Neutron ports, so is agnostic to what is connected to the network in the
form of ports. This leads to a port-oriented approach to Traffic Steering in
Neutron.

Such an approach can allow traffic to be forwarded between Neutron
ports (network services and machines) according to traffic classification.
In this way a tenant can specify a "personalized" chaining sequence according
to the type of traffic.

It is up to the user to instantiate the network service by provisioning a
machine and configuring the network service software in it, making sure it is
attached to a Neutron network. Note that a port can be associated to more than
one chain (and steering) instance allowing a service/VM instance to be part of
multiple chains.

Furthermore, integration with the Group-based Policy Service Chaining
abstraction should be transparent while kept at the user intent level,
capturing application requirements without being overwhelmed by infrastructure
resources. This is a challenge that can probably be solved with notion of
service classes or flavors, to be introduced in Group-based Policy [ref].

The redirect Policy Action of Policy Rules will be reused for Traffic Steering
because it is only a provider for Service Chaining, meaning that the Service
Chaining abstraction is left untouched. The same applies to the Policy
Classifier. It may be beneficial to have traffic classifiers per chain hop,
which would be carried out at the Service Chaining domain instead of the
Group-based Policy domain, but it is out of scope in this blueprint.

A new resource, named Port Chain, will be introduced, which defines how traffic
should be chained/steered amongst ports. In other words, it specifies a list of
source and destination Neutron ports from where traffic should exit and be
steered to. Traffic can be steered to multiple ports in parallel. This resource
will be used by the Traffic Steering provider for Service Chaining.

Service Chain Nodes and Specs will be updated to be compatible with the new
Service Chaining provider. Specifically, Service Chain Nodes require a
different value for the service_type attribute, to support L2 Neutron ports, 
while Service Chain Instances should have a relationship with Port Chains.

Traffic Steering is also being proposed as an API extension. Even though its
abstraction is too low level for the Group-based Policy project, it may be
reused in the future for other steering scenarios. GBP users need not use
Traffic Steering directly. The idea is that the Service Chain provider
associated to Traffic Steering maps its resources to the Traffic Steering API
resources.

In order for Traffic Steering to be materialize, a driver must implement the
port chaining methods, fulfilling the Service Chain in the network. The basic
operation that should be possible at the driver side is to flow specific
traffic from whatever network element is mapped to a Neutron port, to something
else (also mapped to a Neutron port). How and which driver should implement
this functionality is open to discussion.
The two major possibilities we see are:
* Traffic Steering having its own driver for port chaining;
* Traffic Steering relying on whichever drivers are active in the
Service Chaining plugin, for port chaining.

Furthermore, there should be a reference driver implementation for Traffic
Steering, with the following options appearing to be more achievable as of now:
* OpenDaylight, due to their existing efforts on SFC and GBP integration;
* Open vSwitch, due to its quick/easy development effort, enough for a
reference implementation.

Finally, python-gbpclient and group-based-policy-ui (Horizon) will be updated
to reflect the new ability to use generic services via Traffic Steering,
without exposing Traffic Steering or Neutron ports or VMs themselves.

Until the discussion on service classes and flavors for Group-based Policy
reaches a final definition, the implementation associated with this blueprint
will be carried out by exposing Neutron ports to GBP users (when they define
Service Chains via Traffic Steering). This will accelerate a PoC status which
will then evolve to a fully abstracted implementation when the point mentioned
earlier is defined.
Accordingly, this blueprint will also be updated to a final version.


Alternatives
------------

Keep using GBP's Service Chaining service types, which is limited to the
Heat orchestrated Neutron services.


Data model impact
-----------------

A new data model is proposed to persist Traffic Steering definitions, in the
form of Port Chains::

 +---------+       +---------------+
 |  Port   |1     1| Service Chain |
 |  Chain  +-------+   Instance    |
 +---------+       +---------------+

Port Chain (port_chains):

+----------------+-------+---------+---------+------------+----------------+
|Attribute       |Type   |Access   |Default  |Validation/ |Description     |
|Name            |       |         |Value    |Conversion  |                |
+================+=======+=========+=========+============+================+
|id              |string |RO, all  |generated|N/A         |identity        |
|                |(UUID) |         |         |            |                |
+----------------+-------+---------+---------+------------+----------------+
|tenant_id       |string |RO, all  |from auth|N/A         |                |
|                |(UUID) |         |token    |            |                |
+----------------+-------+---------+---------+------------+----------------+
|name            |string |RW, all  |''       |string      |human-readable  |
|                |       |         |         |            |name            |
+----------------+-------+---------+---------+------------+----------------+
|description     |string |RW, all  | ''      |string      |human-relevant  |
|                |       |         |         |            |description     |
+----------------+-------+---------+---------+------------+----------------+
|ports           |dict   |RW, all  |[]       |            |dict of lists   |
|                |(list) |         |         |            |of Neutron ports|
+----------------+-------+---------+---------+------------+----------------+
|override_warning|boolean|R, all   |         |            |                |
+----------------+-------+---------+---------+------------+----------------+

The 'ports' attribute is a dictionary of lists of Neutron ports. Dictionary
keys are Neutron port UUIDs. Dictionary values are list of Neutron port UUIDs.
Ingress traffic from each key port is steered to all ports in the corresponding
value (list of Neutron port UUIDs).

Service Chain Instances will be mapped to Port Chains, when they
correspond to Service Chain Specs based on the Traffic Steering SFC provider.

The API also provides a "override_warnings" flag to alert for situations where
there is more than one path from a source to the same destination (e.g. {'p1':
['p2', 'p3'],'p2': ['p4'], 'p3': ['p4']}), or there is a path where a
destination port can also be a source port and hence forming a loop (e.g.
{'p1': ['p2', 'p3'],'p3': ['p1']}). In case there is overlap we want the user
to know as it may lead to an improper behaviour (e.g. traffic getting doubled).
This is just a warning, since the steering visibility is limited to ports and
the actual behaviour can only be fully understood by taking the actual
services attached to each port into account.


REST API impact
---------------

A new API extension is going to be introduced. Base URL for the Traffic
Steering API is /v2.0/traffic_steering/

The following new resources are being introduced:

.. code-block:: python

  RESOURCE_ATTRIBUTE_MAP = {
      'port_chains': {
          'id': {'allow_post': False, 'allow_put': False,
                 'validate': {'type:uuid': None}, 'is_visible': True},
          'tenant_id': {'allow_post': True, 'allow_put': False,
                        'required_by_policy': True, 'is_visible': True},
          'name': {'allow_post': True, 'allow_put': True,
                   'validate': {'type:string': None}, 'is_visible': True,
                   'default': ''},
          'description': {'allow_post': True, 'allow_put': True,
                          'validate': {'type:string': None},
                          'is_visible': True, 'default': ''},
          'ports': {'allow_post': True, 'allow_put': True,
                    'validate': {'type:dict_of_list_or_none': None},
                    'convert_to': attr.convert_none_to_empty_list,
                    'is_visible': True},
      },
  }


The following new API methods are being introduced:

+-----------+--------------------------------+----------------------+
|Object     |URI                             |Type                  |
+===========+================================+======================+
|Port Chain |/port_chains                    +GET                   |
+-----------+--------------------------------+----------------------+
|Port Chain |/port_chains                    +POST                  |
+-----------+--------------------------------+----------------------+
|Port Chain |/port_chain/{id}                +GET                   |
+-----------+--------------------------------+----------------------+
|Port Chain |/port_chain/{id}                +PUT                   |
+-----------+--------------------------------+----------------------+
|Port Chain |/port_chain/{id}                +DELETE                |
+-----------+--------------------------------+----------------------+


Examples
~~~~~~~~~~~~~~~~~~~

Create Port Chain:

.. code-block:: json

   POST /port_chains.json
   Content-Type: application/json
   Accept: application/json
   {
       "name": "chain1",
       "ports": {
                    "681fedb8-342d-4891-93e5-d9f618c0a2da":
                        ["18a50d57-37bd-44bf-be47-bf921207f203",
                         "3363f633-7d84-46f5-b497-d6b41464ccfc"
                        ],
                    "18a50d57-37bd-44bf-be47-bf921207f203":
                        ["18a50d57-37bd-44bf-be47-bf921207f203",
                         "299b25fd-968d-466b-ab46-bf4667c8cbd5"
                        ],
                }
   }

A concern with this API proposal is the missing classifier resource. In the
previous proposal of Traffic Steering abstraction [1]_, a dedicated Classifier
resource existed. For Group-based Policy, the relevant Policy Classifier can be
used. Comments are welcome regarding classification and whether the API should
be kept or absorved in the existing Service Chain resources.


Security impact
---------------

Describe any potential security impact on the system.  Some of the items to
consider include:

* Does this change touch sensitive data such as tokens, keys, or user data?

No

* Does this change alter the API in a way that may impact security, such as
  a new way to access sensitive information or a new way to login?

No

* Does this change involve cryptography or hashing?

No

* Does this change require the use of sudo or any elevated privileges?

No

* Does this change involve using or parsing user-provided data? This could
  be directly at the API level or indirectly such as changes to a cache layer.

No

* Can this change enable a resource exhaustion attack, such as allowing a
  single API interaction to consume significant server resources? Some examples
  of this include launching subprocesses for each connection, or entity
  expansion attacks in XML.

Yes - passing an unlimited dictionary of ports to the Port Chain resource


Notifications impact
--------------------

None


Other end user impact
---------------------

Horizon and the CLI client will be changed.


Performance Impact
------------------

None


Other deployer impact
---------------------

None


Developer impact
----------------

None


Implementation
==============


Assignee(s)
-----------

Primary assignee:
  igordcard (Igor Duarte Cardoso)

Other contributors:
  snaiksat (Sumit Naiksatam)

Past contributors:
  cgoncalves (Carlos Goncalves)
  joaosoares (Joao Soares)
  bsendas (Bruno Sendas)


Work Items
----------

* API and database
* Generic Service Chain resources, not tied to Heat orchestration, etc.;
* Service Chain provider mapping;
* Traffic Steering driver interface
* OVS / OpenDaylight reference driver
* python-neutronclient
* gbp-ui (Horizon)
* DevStack


Dependencies
============

None


Testing
=======

Driver implementations will have to provide Tempest tests that ensures correct
Traffic Steering rules enforcement. Furthermore, every Work Item must include
respective Unit Tests.


Documentation Impact
====================

None


References
==========

.. [1] Traffic Steering Abstraction for Neutron,
   https://review.openstack.org/#/c/92477/

.. [2] ETSI Network Functions Virtualization (NFV) Industry Specification Group
   (ISG),
   http://www.etsi.org/technologies-clusters/technologies/nfv

.. [3] Mehmet Ersue (ETSI NFV MANO WG Co-chair), "ETSI NFV Management and
   Orchestration - An Overview",
   http://www.ietf.org/proceedings/88/slides/slides-88-opsawg-6.pptx

.. [4] NFV ISG PoC Proposal,
   http://nfvwiki.etsi.org/images/NFVPER%2814%29000011_NFV_ISG_PoC_Proposal_-_Multi-vendor_Distributed_NFV.pdf

.. [5] Service Function Chaining Problem Statement, IETF draft,
   https://tools.ietf.org/html/draft-ietf-sfc-problem-statement-13

.. [6] ServiceBase and Service Insertion,
   https://review.openstack.org/#/c/93128/

.. [7] Advanced Services Common Framework,
   https://docs.google.com/document/d/1fmCWpCxAN4g5txmCJVmBDt02GYew2kvyRsh0Wl3YF2U

.. [8] "Neutron Services' Insertion & Chaining Model, and API" (old doc),
   https://docs.google.com/document/d/1fmCWpCxAN4g5txmCJVmBDt02GYew2kvyRsh0Wl3YF2U

.. [9] Initial version of service chaining specification
   https://review.openstack.org/#/c/93524/