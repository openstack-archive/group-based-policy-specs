..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
L3 Policy Mapping to Address-Scope and Subnet-Pools
===================================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/address-scope-mapping


Problem description
===================

Starting from the Mitaka release, Neutron defines and supports the use of
Address Scopes [#]_. Subnetpools were defined in the earlier Kilo release and
can now be used within Address Scopes. The notion of an address scope has
existed in GBP in the form of L3-Policy. The GBP L3-Policy manages the address
scope and subnet pools with its own home-grown implementation. It would be
preferable to leverage the Neutron Address Scope and Subnetpool instead.


Proposed change
===============

The current mapping of GBP l3_policy to a list of Neutron routers will be
augmented with the mapping to a Neutron address_scope., and an implicit
association with a Neutron subnetpool.

There will be 1-1 mapping between GBP l3_policy and the Neutron address_scope.

A Neutron address_scope resource ID can be provided as an attribute when
creating a GBP l3_policy. The ip_version of the address_scope will be used as
the ip_version of the l3_policy. If an ip_version for the l3_policy is also
provided, but does not match the ip_version of the address_scope, an exception
will be raised. In the explicit model, during creation, the shared value of the
l3_policy has to match that of the address_scope, else an exception will be
raised. If the shared value of the Neutron address_scope is updated, a
corresponding update of the shared value of the l3_policy will be allowed.
(Automatically syncing these two is a separate discussion and outside the scope
of this spec.)

If a Neutron address_scope is not explicitly provided, one will be created
implicitly. In this implicit model, if the l3_policy is shared, the
address_scope will also be shared. If the shared value of the l3_policy is
updated, the shared value of the address_scope will also be automatically
updated accordingly.

In either of the above cases, a Neutron subnetpool will be implicitly created
within the address_scope. The ip_version of the subnetpool will be set to that
of the address_scope. The default_prefixlen, min_prefixlen, and max_prefixlen,
will be set to the subnet_prefix_length of the l3_policy. If a subnetpool
already exists in the address_space, and overlaps with the ip_pool of the
l3_policy, it will be handled at the driver level. In the simplest
implementation, an exception will be raised if there is an overlap. (A more
sophiticated strategy could consider existing subnetpools, and implicitly add
additional subnetpools for the IP ranges that are not covered.)

It can also be evaluated if there is a need to explicitly provide the
subnetpool_id during l3_policy creation. This requires consideration of the
possible combinations of l3_policy, address_scope, and subnetpool and how to
reconcile those when all are provided. This enahancement in outside the scope
of this spec and is slated for future disucssion.


Data model impact
-----------------

The mapping for the l3_policy will be modeled in the DB as follows:

::

 class L3PolicyMapping(gpdb.L3Policy):
     """Mapping of L3Policy to set of Neutron Routers."""
     __table_args__ = {'extend_existing': True}
     __mapper_args__ = {'polymorphic_identity': 'mapping'}
     address_scope_id = sa.Column(
         sa.String(36), sa.ForeignKey('address_scopes.id'), unique=True)
     routers = orm.relationship(L3PolicyRouterAssociation,
                                cascade='all', lazy="joined")

Note that address_scope_id is not nullable. It is either created implicitly or
provided explicitly, but is always required. This is a backward incompatible
change and a script will be provided to migrate data from existing deployments
to this new structure. The script will essentially create an address_scope and
subnetpool for each existing l3_policy.

In addition, additional tables will be added to track the Neutron address_scope
and subnetpool resources created by GBP.

::

 class OwnedAddressScope(model_base.BASEV2):
     """An Address Scope owned by the resource_mapping driver."""

     __tablename__ = 'gpm_owned_address_scopes'
     address_scope_id = sa.Column(sa.String(36),
                                  sa.ForeignKey('address_scopes.id',
                                                ondelete='CASCADE'),
                                  nullable=False, primary_key=True)


 class OwnedSubnetpool(model_base.BASEV2):
     """A Subnetpool owned by the resource_mapping driver."""

     __tablename__ = 'gpm_owned_subnetpools'
     subnetpool_id = sa.Column(sa.String(36),
                               sa.ForeignKey('subnetpools.id',
                                             ondelete='CASCADE'),
                               nullable=False, primary_key=True)


REST API impact
---------------

This is how the udpated l3_Policy mapping would like in terms of the mapping
extension definition

::

    gp.L3_POLICIES: {
        'address_scope_id': {'allow_post': True, 'allow_put': False,
                             'validate': {'type:uuid_or_none': None},
                             'is_visible': True, 'default': None},
        'routers': {'allow_post': True, 'allow_put': True,
                    'validate': {'type:uuid_list': None},
                    'convert_to': attr.convert_none_to_empty_list,
                    'is_visible': True, 'default': None},
    },


Security impact
---------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------

The l3_policy creation workflow has an optional address_scope argument. This
new workflow will be reflected in all clients and UI.


Performance impact
------------------

Better performance is expected on account of the change in the strategy to
allocate subnets that comes with the subnetpool resoure use.

Other deployer impact
---------------------

Deployers need to be aware of the new mapping, both, from an API usage
perspective, and also from debugging and troubleshooting.

Developer impact
----------------

The l3_policy Mapping API changes as indicated before.

Community impact
----------------

Better mapping between GBP and Neutron.


Alternatives
------------

Existing implementation


Implementation
==============

GBP service side implementation will cover updates to the API, DB, implicit,
and resource mapping drivers.

Client will be updated to return the mapped attributes. Updates to UI and Heat
will also be performed as follow up patches.

Assignee(s)
-----------

snaiksat + GBP team


Work items
----------

API, DB, and driver layer updates to GBP Resources.

Dependencies
============

None


Testing
=======

Relevant UTs will be added.

Tempest Tests
-------------

None


Functional Tests
----------------

The exisiting functional tests should cover that there are no regressions.
Some changes might be required to test that the mapped Neutron resources are
created and deleted.


API Tests
---------

UTs


Documentation impact
====================

User Documentation
------------------


Developer Documentation
-----------------------

Devref document will be added.

References
==========

.. [#] http://docs.openstack.org/developer/neutron/devref/address_scopes.html
