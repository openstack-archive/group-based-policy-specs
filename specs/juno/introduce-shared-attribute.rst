..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
Introduce globally shared resources
===================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/introduce-shared-attribute

Today, it's not possible to create shared GBP resources.
This is especially useful in order to avoid duplication of policies
among tenants.

This blueprint introduces a "shared" attribute to certain GBP resources.

Problem description
===================

In the context of concerns separation, it's very important that a user
(e.g. the admin) shares some of the resources he created in order for
different kind of users to be able to consume them.

To achieve this, the API should be able to offer a way to specify
whether a resources is shared or not. This behavior doesn't exist
in our current Group Based Policy implementation.

Proposed change
===============

This change proposes the introduction of a "shared" attribute for the
following GBP resources:

- Contracts;
- Endpoint Groups;
- L2 Policies;
- L3 Policies;
- Network Service policies;
- Policy Rules;
- Policy Classifiers.

The behavior will be consistent with Neutron's already existing
sharing policy. Which means that a given resource can be either
consumable by a single tenant or shared globally.
Shared resources will be modifiable only by the owner or the
admin when applied.

The Endpoint resource has been excluded from the list above
since it is intrinsically something that the user creates and
consumes for himself.


Alternatives
------------

At this time there's no alternative proposal.

Data model impact
-----------------

A "shared" field is added to the resources listed in
the "Proposed change" section.

REST API impact
---------------

The REST API will show the "shared" attribute for the
resource listed in the "Proposed change" section.

Security impact
---------------

This blueprint has no security impact.

Notifications impact
--------------------

This blueprint has no impact on notifications.

Other end user impact
---------------------

The end user will now be able to see and consume
shared resources.

Performance Impact
------------------

This blueprint does not have impact on performance.

Other deployer impact
---------------------

This blueprint does not have deployment impact

Developer impact
----------------

GBP driver's developers should now be aware that some
resources could be shared among tenants and therefore
acting consequently.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mmaleckk

Other contributors:
  None

Work Items
----------

* Add resource attribute to REST API;

* Add model fields to the proper resources;

* Refactor resource_mapping driver to support shared resources.

Dependencies
============

None

Testing
=======

Unit tests will be added to verify the resource visibility
and usability.

Documentation Impact
====================

Eventual GBP documentation will need to provide explanations
on how the "shared" attribute works and examples on how to
use it.

References
==========

None
