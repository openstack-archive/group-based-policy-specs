..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===========================================
Group Based Policy Extensions - Policy Rule
===========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/gbp-policy-rule-extension

Group Based Policy [1] was introduced in the Juno release. However, it has
some limitations which are addressed in this blueprint.



Problem description
===================
The following issues are described in this blueprint:

1. Only one Policy Classifier is allowed per Policy Rule. There is need
for a list of Policy Classifiers instead of a single Policy Classifier
to allow for more flexible selection of packet flows that are to be included
in a Policy Rule. Each Policy Classifier in the list must have precedence
value to define the order of evaluation.

2. Each Policy Rule must have a logic_selector to specify how the 
classifiers are to be combined for matching. If the logic selector is set
to OR then any of classifiers matching will cause the Policy Rule to trigger
and its actions be executed. If the logic selector is set to AND then
all of classifiers must match to cause the Policy Rule to trigger and
its actions be executed.

3. Each Policy Rule in a Policy Contract must have a precedence value
to define the order of execution of actions for the Policy Rules in a
Contract. Currently there is only the implicit order of Policy Rules in
the policy_rules list of a Contract to specify precedence of Policy Rules.

4. Each Policy Action in a Policy Rule must have a precedence value to
define the order of execution of actions in a Policy Rule. Currently
there is only the implicit order of Policy Actions in the action_list
of a Policy Rule to specify precedence of Policy Actions.

5. There is a need to be able to specify that the first Policy Rule in
a Contract that matches is the only Policy Rule whose actions are executed.
A first_match selector can be added to a Contract: if true then only the
actions of the first Policy Rule (in the policy_rules list) in the
Contract will be executed if there is a classifier match for that
Policy Rule; otherwise the actions of other Policy Rules in the
Contract that have a classifier match are also executed.

Proposed change
===============
This work is intended to be done during the Kilo release.

Policy Rule Extensions

The following fields are added to the GBP Policy Rule:
 1. classifier_list replaces classifier
 2. logic_selector
 3. precedence

+---------------+----------+--------+---------+----+-------------------+
|Attribute      |Type      |Access  |Default  |CRUD|Description        |
|Name           |          |        |Value    |    |                   |
+===============+==========+========+=========+====+===================+
|id             |string    |RO, all |generated|R   |identity           |
|               |(UUID)    |        |         |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|tenant_id      |uuid-str  |RO, all |from auth|CR  |Tenant Id          |
|               |          |        |token    |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|name           |string    |RW, all |''       |CRU |human-readable     |
|               |          |        |         |    |name               |
+---------------+----------+--------+---------+----+-------------------+
|description    |string    |RW, all |''       |CRU |human-readable     |
+---------------+----------+--------+---------+----+-------------------+
|contract_filter|uuid-str  |RW, all |None     |CRU |Filter UUID        |
|               |          |        |         |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|classifier_list|List      |RW, all |None     |CRU |List of            |
|               |(uuid-str)|        |         |    |Classifier UUIDs   |
+---------------+----------+--------+---------+----+-------------------+
|logic_selector |Enum      |RW, all |any      |CRU |‘any’, ‘all’       |
|               |          |        |         |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|action_list    |List      |RW, all |None     |CRU |List of            |
|               |(uuid-str)|        |         |    |Action UUIDs       |
+---------------+----------+--------+---------+----+-------------------+
|precedence     |Integer   |RW, all |0        |CRU |Precedence value   |
+---------------+----------+--------+---------+----+-------------------+

C. Policy Action Extensions

To add precedence to the GBP Policy Action

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
+-----------+----------+--------+---------+----+-------------------+
|value      |String    |RW, all |None     |CRU |Action-specific    |
|           |          |        |         |    |value              |
+-----------+----------+--------+---------+----+-------------------+
|precedence |Integer   |RW, all |0        |CRU |Precedence value   |
+-----------+----------+--------+---------+----+-------------------+


D. Contract Extensions

The first_match attribute is added to the GBP Contract.

+---------------+----------+--------+---------+----+-------------------+
|Attribute      |Type      |Access  |Default  |CRUD|Description        |
|Name           |          |        |Value    |    |                   |
+===============+==========+========+=========+====+===================+
|id             |string    |RO, all |generated|R   |identity           |
|               |(UUID)    |        |         |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|tenant_id      |uuid-str  |RO, all |from auth|CR  |Tenant Id          |
|               |          |        |token    |    |                   |
+---------------+----------+--------+---------+----+-------------------+
|name           |string    |RW, all |''       |CRU |human-readable     |
|               |          |        |         |    |name               |
+---------------+----------+--------+---------+----+-------------------+
|description    |string    |RW, all |''       |CRU |human-readable     |
+---------------+----------+--------+---------+----+-------------------+
|policy_rules   |List      |RW, all |Empty    |CRU |List of Policy Rule|
|               |(uuid-str)|        |         |    |UUIDs              |
+---------------+----------+--------+---------+----+-------------------+
|child_contracts|List      |RW, all |None     |CRU |List of Contract   |
|               |(uuid-str)|        |         |    |UUIDs              |
+---------------+----------+--------+---------+----+-------------------+
|first_match    |Bool      |RW, all |false    |CRU |true, false        |
+---------------+----------+--------+---------+----+-------------------+


Alternatives
------------

There are no proposed alternatives to what is proposed in this document.


Data model impact
-----------------

Migration scripts will be added to support the new attributes.


REST API impact
---------------
Changes to group_policy.py are shown in the following sections.


A. New Policy Rule attributes::

    POLICY_RULES: {
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
        'enabled': {'allow_post': True, 'allow_put': True,
                    'default': True, 'convert_to': attr.convert_to_boolean,
                    'is_visible': True},
        'policy_classifier_id': {'allow_post': True, 'allow_put': True,
                                 'validate': {'type:uuid': None},
                                 'is_visible': True, 'required': True},
        'policy_actions': {'allow_post': True, 'allow_put': True,
                           'default': None, 'is_visible': True,
                           'validate': {'type:uuid_list': None},
                           'convert_to': attr.convert_none_to_empty_list},
        'policy_classifiers': {'allow_post': True, 'allow_put': True,
                           'default': None, 'is_visible': True,
                           'validate': {'type:uuid_list': None},
                           'convert_to': attr.convert_none_to_empty_list},                  
    },



Security impact
---------------

Existing security for Group-Based Policy [1] applies.

Notifications impact
--------------------

No change from Group-Based Policy.

Other end user impact
---------------------
The CLI for creating a policy rule with a classifier-list,
logic-selector and precedence::

       neutron gp-policy-rule-create
           -–classifiers “22034455989843”, “12436bce978970”
           --logic-selector “any”
           --precedence “2”


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

.. [1] Group-based policy https://review.openstack.org/#/c/87825
