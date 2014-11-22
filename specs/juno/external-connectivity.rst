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

**Nat Policy** A pool of IP addresses (range/cidr) that will be used
by the drivers to implemented NAT capabilities when needed. How
and whether the Nat Policy will be used by the reference implementation
is out of scope of this blueprint.

**External Access Segment** A combination of a cidr and an address
representing the L3 policy interface to a given portion of the
external world. The L3 Policy needs to provide which address it has to
expose on a given external access segment.

**External Access Route** A combination of a cidr and a next hop
representing a portion of the external world reachable by the L3_Policy
via a given next hop

**External Access Policy** A collection of EASs that provides and
consumes PRSs in order to define the data path filtering for the
north-south traffic.

Example CLI:

Coming Soon

Alternatives
------------

Today there's no GBP alternative for external connectivity.

Data model impact
-----------------

New model is introduced in order to represent the external
connectivity::

 +----------+
 | External |
 | Access   |
 | Policy   |
 +----+-----+
      |m
      |
      |n
 +----+-------+          +---------+
 | Ext. Access|1        m| NAT     |
 | Segment    +----------+ Policy  |
 +----+-------+          +---------+
      |
      |                  +------------+
      |1                n| Ext. Access|
      +------------------+ Route      |
      |                  +------------+
      |
      |                  +------------+
      |1               n | L3P Address|
      +------------------+ Allocation |
                         +------------+

All objects (excluded EAR and L3PAA) have the following common attributes:
  * id - standard object uuid
  * name - optional name
  * description - optional annotation
  * shared - whether the object is shared or not

External Access Segment
  * ip_version - [4, 6]
  * address_cidr - string on the form <my_ip>/<prefix_length> which describes
    a specific address exposed by the external policy in a specific subnet
  * encap_type - string, encapsulation type on this segment
  * encap_value - string, encapsulation value on this segment
  * l3_policies - a list of l3_policies UUIDs
  * port_address_translation - boolean, specifies whether PAT needs to be performed
    using the addresses allocated for the L3P

NAT Policy
  * ip_version - [4,6]
  * ip_pool - string, IPSubnet with mask used to pull addresses from
    for NAT purposes
  * external_access_segments - UUID list of the EASs using this NAT policy

External Access Route
  * cidr - string, IPSubnet with mask used to represent a portion of the
    external world
  * netx_hop - string, ip address describing where the traffic should be sent
    in order to reach cidr
  * external_access_segment_id - UUID of the EAS through which this EAR is
    consumable

External Access Policy
  * external_access_segments - a list of external access segments UUIDs
  * provided_policy_rules_set - a list of provided policy rules set UUIDs
  * consumed_policy_rules_set - a list of consumed policy rules set UUIDs

L3P Address Allocation
  * external_access_segment_id - EAS UUID
  * l3_policy_id - L3P UUI
  * allocated_address - IP address belonging to the EAS subnet

The following tables will be modified:

L3 Policy
  * (add column) external_access_segments - list of EAS UUIDs

REST API impact
---------------

Code snippet describing the new model::

    EXTERNAL_ACCESS_POLICIES: {
        'id': {'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None},
               'is_visible': True, 'primary_key': True},
        'name': {'allow_post': True, 'allow_put': True,
                 'validate': {'type:string': None},
                 'default': '', 'is_visible': True},
        'description': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:string': None},
                        'is_visible': True, 'default': ''},
        'tenant_id': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:string': None},
                      'required_by_policy': True, 'is_visible': True},
        'external_access_segments': {
            'allow_post': True, 'allow_put': True, 'default': None,
            'validate': {'type:uuid_list': None},
            'convert_to': attr.convert_none_to_empty_list, 'is_visible': True},
        'provided_policy_rule_sets': {'allow_post': True, 'allow_put': True,
                                      'validate': {'type:dict_or_none': None},
                                      'convert_to':
                                      attr.convert_none_to_empty_dict,
                                      'default': None, 'is_visible': True},
        'consumed_policy_rule_sets': {'allow_post': True, 'allow_put': True,
                                      'validate': {'type:dict_or_none': None},
                                      'convert_to':
                                      attr.convert_none_to_empty_dict,
                                      'default': None, 'is_visible': True},
    },
    EXTERNAL_ACCESS_SEGMENTS: {
        'id': {'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None},
               'is_visible': True, 'primary_key': True},
        'name': {'allow_post': True, 'allow_put': True,
                 'validate': {'type:string': None},
                 'default': '', 'is_visible': True},
        'description': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:string': None},
                        'is_visible': True, 'default': ''},
        'tenant_id': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:string': None},
                      'required_by_policy': True, 'is_visible': True},
        'ip_version': {'allow_post': True, 'allow_put': False,
                       'convert_to': attr.convert_to_int,
                       'validate': {'type:values': [4, 6]},
                       'default': 4, 'is_visible': True},
        'address_cidr': {'allow_post': True, 'allow_put': False,
                         'validate': {'type:subnet': None},
                         'default': attr.ATTR_NOT_SPECIFIED,
                         'is_visible': True},
        'encap_type': {'allow_post': True, 'allow_put': True,
                       'validate': {'type:string': None},
                       'default': attr.ATTR_NOT_SPECIFIED,
                       'enforce_policy': True,
                       'is_visible': True},
        'encap_value': {'allow_post': True, 'allow_put': True,
                        'convert_to': attr.convert_to_int,
                        'enforce_policy': True,
                        'default': attr.ATTR_NOT_SPECIFIED,
                        'is_visible': True},
        'external_access_policies': {
            'allow_post': False, 'allow_put': False, 'default': None,
            'validate': {'type:uuid_list': None},
            'convert_to': attr.convert_none_to_empty_list, 'is_visible': True},
        'external_access_routes': {
            'allow_post': True, 'allow_put': True,
            'default': attr.ATTR_NOT_SPECIFIED,
            'validate': {'type:hostroutes': None},
            'is_visible': True},
        'l3_policies': {
            'allow_post': False, 'allow_put': False, 'default': None,
            'validate': {'type:uuid_list': None},
            'convert_to': attr.convert_none_to_empty_list, 'is_visible': True},
        'port_address_translation': {
            'allow_post': True, 'allow_put': True,
            'default': False, 'convert_to': attr.convert_to_boolean,
            'is_visible': True, 'required_by_policy': True,
            'enforce_policy': True},
    },
    NAT_POLICIES: {
        'id': {'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None},
               'is_visible': True, 'primary_key': True},
        'name': {'allow_post': True, 'allow_put': True,
                 'validate': {'type:string': None},
                 'default': '', 'is_visible': True},
        'description': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:string': None},
                        'is_visible': True, 'default': ''},
        'tenant_id': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:string': None},
                      'required_by_policy': True, 'is_visible': True},
        'ip_version': {'allow_post': True, 'allow_put': False,
                       'convert_to': attr.convert_to_int,
                       'validate': {'type:values': [4, 6]},
                       'default': 4, 'is_visible': True},
        'ip_pool': {'allow_post': True, 'allow_put': False,
                    'validate': {'type:subnet': None},
                    'default': attr.ATTR_NOT_SPECIFIED,
                    'is_visible': True},
        'external_access_segment_id': {'allow_post': True, 'allow_put': True,
                                       'validate': {'type:uuid': None},
                                       'is_visible': True, 'required': True},
    }

The following have been modified (only new attributes shown)::

    L3_POLICIES: {
        'external_access_segments': {
            'allow_post': True, 'allow_put': True,
            'validate': {'type:external_access_dict': None},
            'convert_to': attr.convert_none_to_empty_dict,
            'default': attr.ATTR_NOT_SPECIFIED, 'is_visible': True},
    },

More information about the attribute types follows:

**type:hostroutes**
A dictionary in the form {"destination": <cidr>, "nexthop": <ip_address>}

**type:external_access_dict**
A dictionary in the form {<eas_uuid>: [<my_eas_ip>, ...]}. It represents
which EAS the L3P is connected through, and which addresses it uses on it.

Security impact
---------------

Policy Targets within the cloud can be reach and can reach the outside world.
The security implications depend on the way the PRS are composed
by the cloud admin.
In order to talk to the external world, a given Policy Target Group
needs to satisfy the followings:

- The L3P it belongs to must have at least one external access segment and one IP allocated;
- The External Access Segment must have at least one route;
- the External Access Segment must have an External Access Policy;
- The PTG must provide/consume a PRS provided/consumed by the said EAP;
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
