..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.
 http://creativecommons.org/licenses/by/3.0/legalcode
 
==========================================
GBP Floating IP Policy
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/external-connectivity


Problem description
===================

The existing GBP apis do not provide a way of defining a policy where all PTs in a PTG require a Floating IP to be associated with them.

Proposed change
===============

The Network Service Policy resource in GBP, at present supports allocating a single IP address from a given PTG subnet. This is achieved by creating a Network Service Parameters dictionary with type ip_single and value self_subnet to indicate that a single IP Address has to be allocated from the PTG subnet to which the Network Service Policy is attached.
This network_service_params in Network Service Policy will be extended to add a new type ip_pool and value external_segment to represent that all the PTs in the PTG should be allocated a Floating IP.

Reference Implementation:
When a PT is created, the Resource Mapping Driver shall retrieve the Network Service Policy defined for the PTG. If there is a Network Service Parameter with type ip_pool and value external_segment, the external segment is retrieved from the L3 Policy in use for the PTG. Then a Floating IP is allocated and associated with the port associated with the PT.

Network Service Policy create or update operations shall raise an error if an external segment is not associated with the L3Policy for the PTGs the NSP refers to.

Alternatives
------------

All PTGs on a L3Policy that has a external segment associated shall get floating IP associated with all the PTs without the need for a Network Service Policy. The main disadvantage with this approach is that there is no fine grained control with the Floating IP association with implicit external segment association with a L3Policy, in which case all the PTs will end up getting a Floating IP.

Data model impact
-----------------

None.

REST API impact
---------------

None

Security impact
---------------

Policy Targets within the cloud reach the external world and outside world can reach
the Policy Targets when they get a FIP.
The security implications depend on the way the PRS are composed by the cloud admin.
In order to talk to the external world, a given Policy Target Group
needs to satisfy the following:
- The L3P it belongs to must have at least one external access segment and one IP allocated;
- The External Segment must have at least one route;
- the External Segment must have an External Policy;
- The PTG must provide/consume a PRS provided/consumed by the said EP;
- The traffic has to satisfy the filtering rules defined in the PRS;

Notifications impact
--------------------
This blueprint has no impact on notifications.

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
 Magesh GV (magesh-gv)

Other contributors:
  None

Work Items
----------

- Database and API;
- Neutron mapping driver;

Dependencies
============

None

Testing
=======

New unit tests will be added for the floating IP model, and existing
unit tests for the mapping will be updated when needed.

Documentation Impact
====================

Eventual GBP documentation will need to address configuration
of Network Service Policy to associate floating IP.

References
==========

None
