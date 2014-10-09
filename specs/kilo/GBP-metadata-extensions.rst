..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Group Based Policy Extensions - Metadata
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/gbp-metadata-extension

Group Based Policy [1] was introduced in the Juno release. However, it has
some limitations which are addressed in this blueprint.



Problem description
===================
There is a need for metadata insertion as an action within a Policy Rule.
There is work underway within IETF SFC working group to allow packets to
transport metadata in the service chain header for delivery to service
functions in a service chain.



Proposed change
===============
This work is intended to be done during the Kilo release.

Policy Action Extensions

The following changes are made to the GBP Policy Action:

 1. insert-metadata action

+-----------+----------+--------+---------+----+-------------------+
|Attribute  |Type      |Access  |Default  |CRUD|Description        |
|Name       |          |        |Value    |    |                   |
+===========+==========+========+=========+====+===================+
|id         |string    |RO, all |generated|R   |identity           |
|           |(UUID)    |        |         |    |                   |
+-----------+----------+--------+---------+----+-------------------+
|tenant_id  |uuid-str  |RO, all |from auth|CR  |Tenant Id          |
|           |          |        |token    |    |                   |
+-----------+----------+--------+---------+----+-------------------+
|name       |string    |RW, all |''       |CRU |human-readable     |
|           |          |        |         |    |name               |
+-----------+----------+--------+---------+----+-------------------+
|description|string    |RW, all |''       |CRU |human-readable     |
+-----------+----------+--------+---------+----+-------------------+
|type       |Enum      |RW, all |Allow    |CRU |Action type        |
|           |          |        |         |    |'allow', 'redirect'|
|           |          |        |         |    |'insert-metadata'  |
+-----------+----------+--------+---------+----+-------------------+
|value      |String    |RW, all |None     |CRU |Action-specific    |
|           |          |        |         |    |value              |
+-----------+----------+--------+---------+----+-------------------+



Alternatives
------------

There are no proposed alternatives to what is proposed in this document.


Data model impact
-----------------

Migration scripts will be added to support the new attributes.


REST API impact
---------------
Changes to group_policy.py to add an Insert Metadata action::

  gp_supported_actions = [None, gp_constants.GP_ALLOW, gp_constants.GP_REDIRECT,
                          gp_constants.GP_INSERT_METADATA]



Security impact
---------------
Existing security for Group-Based Policy [1] applies.

Notifications impact
--------------------
No change from Group-Based Policy.

Other end user impact
---------------------
The CLI for creating an insert-metadata action::

       neutron gp-policy-action-create 
           -–type "insert-metadata" -–value "application-id:22034455989843"

Changes will be made to python-neutronclient to support this CLI.
Changes will also be made to Horizon and Heat.

Performance Impact
------------------

No significant performance impact.


Other deployer impact
---------------------
No other impact.

Developer impact
----------------
None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Louis Fourie (louis.fourie@huawei.com)

Other contributors:
 Cathy Zhang
 Nicolas Bouthors


Work Items
----------
The work items include:

 1. Implement CRUD operations to specify a new fields.
 2. Implement Database updates to support that API.
 3. Implement Migration scripts to support the new DB items.
 4. Implement python-neutronclient changes for CLI.
 5. Implement Horizon changes.


Dependencies
============

This blueprint depends on Group Based Policy [1].


Testing
=======

Unit tests will be provided.


Documentation Impact
====================
A description of the new attributes must be added to the documentation.


References
==========

[1] Group-based policy https://review.openstack.org/#/c/87825


