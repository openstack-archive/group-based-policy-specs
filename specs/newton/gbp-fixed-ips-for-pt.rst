..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Allow specifying fixed IP addresses for PT creation
===================================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/fixed-ips-for-pt

Problem description
===================

In certain cases it is required to create Policy Targets with specific IP
addresses. This is possible today by first creating a Neutron Port with the
fixed IP(s) and then creating a PT using that Port. It will be desirable to
provide the IP address as an optional attribute when creating the PT itself to
avoid this two-step process.

Proposed change
===============

It is proposed to add a new optional attribute to the policy_target resource.
This new attribute will be modeled after the fixed_ips attribute [#]_ that is used
for Neutron Port.

We have been already using fixed_ips for PT creation in internal calls [#]_. So
the change required is only to the API.

Data model impact
-----------------

There is no change to the data model.

REST API impact
---------------

A new optional attribute "fixed_ips" is added to the group_policy_mapping
extenaion for the policy_target resource:

        'fixed_ips': {'allow_post': True, 'allow_put': True,
                      'default': attr.ATTR_NOT_SPECIFIED,
                      'convert_list_to': attr.convert_kvp_list_to_dict,
                      'validate': {'type:fixed_ips': None},
                      'enforce_policy': True,
                      'is_visible': True},


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


Developer impact
----------------


Community impact
----------------


Alternatives
------------


Implementation
==============


Assignee(s)
-----------

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
.. [#] https://github.com/openstack/neutron/blob/78fff41ee3c56908e96cc302ffcb02c48e84f68a/neutron/api/v2/attributes.py#L196-L202
.. [#] https://github.com/openstack/group-based-policy/commit/0d5cedc413fc99461b39f2854475416679e4e2c1
