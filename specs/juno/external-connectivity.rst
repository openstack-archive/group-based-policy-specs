..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
GBP External Connectivity
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/external-connectivity

Today, the GBP model only represents east-west traffic policies.
This blueprint introduces new API to allow north-south traffic in
a GBP enabled cloud


Problem description
===================

The existing API doesn't provide any way to represent external access
(i.e. north-south traffic) in a GBP enabled cloud.
In order to obtain such capabilities, today the user has to go to
Neutron and define the right topology by creating external networks
and attaching the proper router interfaces to them.
Furthermore, the above workaround assumes that a Neutron mapping is
in place, which is not true for all the GBP drivers.

Proposed change
===============

This proposal presents new API objects in order to model the external
connectivity policy. Although the objective is always to capture
the user's intent, it has to be noted that this particular case usually
requires a lot of manual configuration by the admin *outside* the cloud
boundaries (e.g. configuring external router), which means that the
usual automation provided by GBP has to be paired with meaningful tools
which allow detailed configuration when needed.

The following new terminology is introduced:

**External Access Policy** A collection of policies that ultimately
describes how a L3_Policy can connect to the outside world.

**Nat Policy** A pool of IP addresses (range/cidr) that will be used
by the drivers to implemented NAT capabilities when needed. How
and whether the Nat Policy will be used by the reference implementation
is out of scope of this blueprint.

**External Access Segment** A combination of a cidr and an address
representing the L3 policy interface to a given portion of the
external world. This tells the L3 Policy which address it has to
expose on a given external subnet.

**External Access Route(???)** A combination of a cidr and a next hop
representing a portion of the external world reachable by the L3_Policy
via a given next hop

**External Policy Group** A collection of EAPs that provides and
consumes contracts in order to define the data path filtering for the
north-south traffic.

Example CLI:

TODO

Alternatives
------------

Today there's no GBP alternative.

Data model impact
-----------------

New model is introduced in order to represent the external
connectivity::

 +----------+
 | External |
 | Policy   |
 | Contract |
 +----+-----+
      |1
      |
      |n
 +----+-------+          +---------+
 | Ext. Access|n        m| NAT     |
 | Policy     +----------+ Policy  |
 +----+-------+          +---------+
      |
      |                  +------------+
      |n                m| Ext. Access|
      +------------------+ Segment    |
      |                  +------------+
      |
      |                  +------------+
      |n                m| Ext. Access|
      +------------------+ Route      |
                         +------------+

All objects have the following common attributes:
  * id - standard object uuid
  * name - optional name
  * description - optional annotation
  * shared - whether the object is shared or not

External Access Policy
  * epc_id - UUID of the External Policy Contract
  * nat_policies - a list of NAT policies UUIDs
  * external_segments - a list of external segments UUIDs
  * external_routes - a list of external routes UUIDs
  * l3_policies - a list of l3_policies UUIDs

NAT Policy
  * ip_pool - string, IPSubnet with mask, used to pull addresses from
    for NAT purposes
  * external_access_policies - UUID list of the external policies using
    this NAT policy

External Access Segment
  * address_cidr - string on the form <my_ip>/<prefix_length> which describes
    a specific address exposed by the external policy in a specific subnet
  * external_access_policies - UUID list of the external policies using this
    EAS

External Access Target
  * cidr - string, IPSubnet with mask, used to represent a portion of the
    external world
  * netx_hop - string, ip address describing where the data should be sent
    in order to reach cidr
  * external_access_policies - UUID list of the external policies using this
    EAT

External Policy Contract
  * provided_policy_rules_set - a list of provided policy rules set UUIDs
  * consumed_policy_rules_set - a list of consumed policy rules set UUIDs

The following tables will be modified:

L3 Policy
  * (add column) external_access_policies -  UUID list of the external policies
    using this L3P

REST API impact
---------------

The objects described in the section above are being added to the REST
interface (TODO: python code snippet)

Security impact
---------------

Policy Targets within the cloud can be reach and can reach the outside world.
The security implications depend on the way the PRS  are composed
by the cloud admin.
In order to talk to the external world, a given Policy Target Group
needs to satisfy the followings:

- The L3P it belongs to must have at least on external policy;
- The External Access Policy must have at least one external segment and route;
- the External Access Policy must have an External Policy Group;
- The PTG must provide/consume a PRS provided/consumed by the EPG;
- The traffic has to satisfy the filtering rules defined in the PRS;

Notifications impact
--------------------

This blueprint has no impact on notifications.

Other end user impact
---------------------

The python client and the UI have to expose the new model
to the end user.

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
 Ivar Lazzaro (mmaleckk)

Other contributors:
  None

Work Items
----------

TBD

Dependencies
============

None

Testing
=======

New unit tests will be added for the external connectivity extension
itself, and existing unit tests for the mapping will be updated
when needed.

Documentation Impact
====================

Eventual GBP documentation will need to address configuration
of external access policy

References
==========

None
