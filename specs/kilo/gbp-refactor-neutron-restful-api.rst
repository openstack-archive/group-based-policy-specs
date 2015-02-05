..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Group Based Policy Refactor: Use Neutron RESTful APIs
=====================================================

Launchpad blueprint:
https://blueprints.launchpad.net/group-based-policy/+spec/neutron-rest-api-refactor

This blueprint proposes using neutron RESTful APIs in resource mapping driver.

Problem description
===================
The current (Kilo) GBP interacts with neutron directly through neutron
internal APIs. This tight coupling prevents GBP from being instantiated
as separate process/service. This blueprint proposes a loose coupling by
moving GBP to use the neutron RESTful APIs. More specifically, a neutron
RESTful API client wrapper will be implemented, and the neutron internal
APIs previously used in GBP will be replaced by calls to this client library.

Proposed change
===============
The proposed change will:

1. Add neutron v2 API module. This module will provide APIs to neutron
constrcuts' CRUD operation. This code will be added to:
gbpservice/network/neutronv2
This will be similar to how nova is doing it [2].
2. Refactor resource mapping driver code to replace neutron direct calls
with the neutron v2 API calls.

Alternative
------------
None.

Data model impact
-----------------
Re-factoring the resource mapping driver code with the neutron RESTful APIs
are invisible to users, therefore should not by itself require structural
changes to the data model currently defined in the group based policy.

REST API impact
---------------
None.

Security impact
---------------
None.

Notifications impact
--------------------
None.

Other end user impact
---------------------
None.

Performance Impact
------------------
There will be some minimal performance imapct after refactoring as RESTful
APIs are used.

Other deployer impact
---------------------
None.

Developer impact
----------------
None.

Implementation
==============

Assignee(s)
-----------
Yapeng Wu
Yi Yang

Work Items
----------
1. Add neutron v2 API module.
2. Replace directly neutron calls with neutron v2 API calls in resource
   mapping driver.

Dependencies
============
Neutron Python Client (2.3.9) and plan to move it to 2.3.10 once the gbpclient
also moves its dependency up to 2.3.10.

Testing
=======
Additional UT for the neutron rest API client will be provided.
Existing UT for the resource mapping driver might need to be updated.

Documentation Impact
====================
None

References
==========
[1]
http://git.openstack.org/cgit/stackforge/group-based-policy-specs/tree/specs/juno/group-based-policy-abstraction.rst
https://wiki.opendaylight.org/view/Group_Policy:Main
[2] https://github.com/openstack/nova/tree/master/nova/network/neutronv2
