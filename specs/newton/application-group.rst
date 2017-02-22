..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=====================================================
Grouping of PTGs based on Application Characteristics
=====================================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/application-group

Problem description
===================

A Policy Target Group (PTG) can be used to represent a specific tier of an
Application (e.g. DB tier). However, there is no construct in GBP that
corresponds to the application itself. Such a construct can help to abstract
the properties of the application (across the PTGs) futher simplifying the
intent specification and automated orchestration.

Proposed change
===============

It is proposed to add a new resource Application Policy Group (APG) that will
have a 1 to many relationship with PTGs. Each PTG will belong to exactly one
APG. The APG does not have relationship (or constraints) with any other GBP
resources.

Each tenant will have a default APG. If no APG is specified, a PTG will be
placed in this default APG. The APG association of the PTG can be updated.
APG will support the shared attribute and can have visibility beyond the
project which owns it.

Data model impact
-----------------

A new table for APG will be added:

::

  class ApplicationPolicyGroup(models_v2.HasId, models_v2.HasTenant):
      name = sa.Column(sa.String(255))
      description = sa.Column(sa.String(255))
      status = sa.Column(sa.String(length=16), nullable=True)
      status_details = sa.Column(sa.String(length=4096), nullable=True)
      policy_target_groups = orm.relationship(PolicyTargetGroup,
                                              backref='application_policy_group')

A new foreign key constraint will be added to the PolicyTargetGroup table:

::

      application_policy_group_id = sa.Column(sa.String(36),
                                              sa.ForeignKey('gp_application_policy_groups.id'),
                                              nullable=True)

REST API impact
---------------

A new resource application_policy_groups will be added:

::

    APPLICATION_POLICY_GROUPS: {
        'id': {'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None}, 'is_visible': True,
               'primary_key': True},
        'name': {'allow_post': True, 'allow_put': True,
                 'validate': {'type:gbp_resource_name': None},
                 'default': '', 'is_visible': True},
        'description': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:string': None},
                        'is_visible': True, 'default': ''},
        'tenant_id': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:string': None},
                      'required_by_policy': True, 'is_visible': True},
        'status': {'allow_post': False, 'allow_put': False,
                   'is_visible': True},
        'status_details': {'allow_post': False, 'allow_put': False,
                           'is_visible': True},
        'policy_target_groups': {'allow_post': False, 'allow_put': False,
                                 'validate': {'type:uuid_list': None},
                                 'convert_to': attr.convert_none_to_empty_list,
                                 'default': None, 'is_visible': True},
        attr.SHARED: {'allow_post': True, 'allow_put': True,
                      'default': False, 'convert_to': attr.convert_to_boolean,
                      'is_visible': True, 'required_by_policy': True,
                      'enforce_policy': True},
    },


A new optional attribute "application_policy_group_id" is added to the
policy_target_group resource:

::

        'application_policy_group_id': {'allow_post': True, 'allow_put': True,
                                        'validate': {'type:uuid_or_none': None},
                                        'default': '', 'is_visible': True}

The client will have to be updated, and CLI for APG will have to be provided.


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

The policy driver interface will be updated to support CRUD operations for APG.


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
