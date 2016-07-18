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

Starting in the Mitaka release Neutron defines and supports the use of Address
Scopes [#]_. Subnetpools were defined in the earlier Kilo release and can now
be used within Address Scopes. The notion of an address scope has existed in
GBP in the form of L3-Policy. The GBP L3-Policy manages the address scope and
subnet pools with its own home-grown implementation. It would be preferable to
leverage the Neutron Address Scope and Subnetpool instead.

Proposed change
===============

The current mapping of GBP l3_policy to a list of Neutron routers will be
augmented with the mapping to a Neutron subnetpool.

There will be 1-1 mapping between GBP l3_policy and the Neutron subnetpool. A
Neutron address_scope and subnetpool will be implicitly created for a
l3_policy if the subnetpool is not explicitly specified via the API.

In ths implicit model, if the l3_policy is shared, the address_scope will also
be shared.

Data model impact
-----------------

The mapping for the L3-Policy will be modeled in the DB as follows:

::

 class L3PolicyMapping(gpdb.L3Policy):
     """Mapping of L3Policy to set of Neutron Routers."""
     __table_args__ = {'extend_existing': True}
     __mapper_args__ = {'polymorphic_identity': 'mapping'}
     subnetpool_id = sa.Column(
         sa.String(36), sa.ForeignKey('subnetpools.id'), unique=True)
     routers = orm.relationship(L3PolicyRouterAssociation,
                                cascade='all', lazy="joined")

Note that subnetpool_id is not nullable. It is either created implicitly or
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

This is how the udpated L3-Policy mapping would like in terms of the mapping
extension definition

::

    gp.L3_POLICIES: {
        'subnetpool_id': {'allow_post': True, 'allow_put': False,
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


Performance impact
------------------

Better performance is expected on account of the change in the strategy to
allocate subnets that comes with the subnet-pool resoure use.

Other deployer impact
---------------------

Deployers need to be aware of the new mapping, both, from an API usage
perspective, and also from debugging and troubleshooting.

Developer impact
----------------

The L3-Policy Mapping API changes as indicated before.

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
