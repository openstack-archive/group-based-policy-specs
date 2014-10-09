..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Refactor mapping extension to expose simplified model
=====================================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/mapping-extension-refactor

As a new API abstraction, GBP needs to define what is the "common denominator"
across present and future drivers in order to expose to the user a model
which is as much useful and semplified as possible, so that scripts that
are valid against the reference implementation can actually be reused
against the drivers without changing the basic expectation. Any new driver
can then add functionalities through extensions, but this is covered
separately.

Problem description
===================

The GBP mapping extension expose to the user a standard way of mapping
GBP resources to Neutrons'. As is today, the mapping extension consists
of:

- Policy Target mapped to Port: R/W access;
- Policy Tartget Group mapped to Subnet: R/W access;
- L2_Domain mapped to Network: R/W access;
- L3_Domain mapped to Router: R/W access.

The problem with the current model is that it forces any new driver to
follow that very specific mapping. Any different mapping implementation
will change the user's expectation and may not work with scripts written
for the reference implementation. This is in fact not needed for multiple
reasons:

- The only thing that GBP always need is a Neutron Port;
- A Neutron Port needs a Subnet and a Network, but they can be
  created implicitly;
- There are very few to None cases in which the user may want to
  manually provide a Subnet or a Network to map GBP resources;
- As long as the Port is known, the rest of the model can be
  retrieved from there;
- A user cares more about restricting connectivity rather than
  specifying the method used to restrict the connectivity.

Proposed change
===============

The existing mapping will be simplified to the following:

- Policy Target mapped to Port: Read Only;
- Policy Tartget Group will have no explicit mapping;
- L2_Domain will have no explicit mapping;
- L3_Domain will have no explicit mapping.

Note that the GBP model will still exist in the API! but the mapping is
not exposed.

The above is all that is needed in order to make GBP work with Nova,
and still covers most of the use cases (eg. it's not very likely for a
user to manually specify a Network or a Router). The advantage, besides
simplicity, is that all the future drivers can use this as a common base so
that the final user doesn't need to change his scripts when going from a
driver to another.

The current reference implementation (resource_mapping) will NOT change
for now.
The only difference is that those extra mappings (Network/Subnets/Routers)
will not be exposed.

Alternatives
------------

Leaving things as they are.

Data model impact
-----------------

the grouppolicy_mapping_db will need a slight refactor. This is because
today we extend existing tables in order to add the mapping
(eg. Policy Target Group table is extended to add a Subnet column).
This approach is not very desiderable because any driver with a
different idea of mapping should not extend existing tables.
Instead, new association tables will be created and correlation
done through join.

REST API impact
---------------

(AS IS)Current group policy mapping extension:

gp.ENDPOINTS: {
    'port_id': {'allow_post': True, 'allow_put': False,
                'validate': {'type:uuid_or_none': None},
                'is_visible': True, 'default': None}}

gp.ENDPOINT_GROUPS: {
    'subnets': {'allow_post': True, 'allow_put': True,
                'validate': {'type:uuid_list': None},
                'convert_to': attr.convert_none_to_empty_list,
                'is_visible': True, 'default': None}}

gp.L2_POLICIES: {
    'network_id': {'allow_post': True, 'allow_put': False,
                   'validate': {'type:uuid_or_none': None},
                   'is_visible': True, 'default': None}}

gp.L3_POLICIES: {
    'routers': {'allow_post': True, 'allow_put': True,
                'validate': {'type:uuid_list': None},
                'convert_to': attr.convert_none_to_empty_list,
                'is_visible': True, 'default': None}}

(TO BE):

gp.ENDPOINTS: {
    'port_id': {'allow_post': False, 'allow_put': False,
                'validate': {'type:uuid_or_none': None},
                'is_visible': True, 'default': None}}


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None since GBP hasn't cut any release yet.

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
  mmaleckk (ivar-lazzaro)

Other contributors:
  gbp core team

Work Items
----------

- API refactor;
- DB compliance.


Dependencies
============

None

Testing
=======

UTs

Documentation Impact
====================

None

References
==========

IRC discussion:
http://eavesdrop.openstack.org/meetings/networking_policy/2014/networking_policy.2014-10-09-18.01.log.html

