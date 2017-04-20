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
augmented with the mapping to a Neutron address_scope (one each for IPv4 and
IPv6) and subnetpools. This spec attempts to lay the foundation for the use
of IPv6 and dual-stack (both IPv4 and IPv6) in the l3_policy. The API and DB
model constructs to support this are addressed in this spec.

The ip_version attribute of GBP's l3_policy is extended to allow specifying
dual-stack support. Currently allowed values for ip_version are 4 for IPv4
and 6 for IPv6. Either 46 or 64 may now be used to indicate dual-stack support.

For each address family (IPv4 and/or IPv6) supported by an l3_policy, there
will be a 1-1 mapping between GBP l3_policy and the Neutron address_scope,
and a 1-many mapping between GBP l3_policy and Neutron subnetpools. The 1-many
mapping allows for use cases where the l3_policy (and address_scope) is shared
across tenants, while the subnetpool is created per tenant and not shared.

Within an address family, the subnetpools that are explicitly associated with
the l3_policy must all belong to the same address_scope. GBP will allocate
subnets only from the subnetpools that are associated with the corresponding
l3_policy. The list of subnets associated with the l3_policy can be updated
by adding or removing subnetpools (subnetpools which are currently being used
cannot be removed).

The implicit subnetpools extension is used to control the desired behavior
when an implicit workflow is used with L3 policies [#]_. Implicit subnetpools
are only used when the ip_version requests a given address family (IPv4 or
IPv6) and the user hasn't explicitly provided a subnetpool and/or address
scope for that address family. The existing implicit subnetpool semantics
must be amended to account for a subnetpool per address family:

  - Only one implicit subnetpool *per address family* can exist *per tenant*
    (if not shared)
  - Only one shared implicit subnetpool *per address family* can exist in the
    whole system.  This doesn't affect private implicit subnetpools

Two properties of address scopes are address namespace and routability.
Address namespace simply means that addresses are guaranteed to be unique
within an address scope. Routability means that neutron routers may only
forward between interfaces that have subnets which were allocated from
subnetpools with the same address scope (note: forwarding is still possible
via NAT when they aren't in the same scope). It's worth pointing out that
since address scopes can span an l3_policy, the address uniqueness is
applied across the l3_policies, but routability is only applied within
each l3_policy (because neutron routers are tied to a single l3_policy)..

Let's consider the various combinations that are possible based on the choice
of implicit and explicit arguments.

#. A l3_policy is created with an ip_pool, ip_version, and
   subnet_prefix_length but no address_scope or subnetpool(s) are provided.
   There are two possibilities in this scenario:

     *  An implicit subnetpool exists for the given ip_version
        In this case the implicit subnetpool and its address scope are
        associated witht he l3_policy.
     *  An implicit subnetpool does not exist for the given ip_version
        In this case an address_scope with the address family of the ip_version
        will be created, and a subnetpool with the ip_pool CIDR will be created
        in that address_scope.

   In both cases, if the l3_policy is shared, the implcitily created
   address_scope and subnetpool will also be shared. The ip_version of the
   l3_policy will be used for the address_scope and the subnetpool. The
   default_prefixlen, will be set to the subnet_prefix_length of the l3_policy.
   (for IPv4, the min_prefixlen and max_prefixlen of the subnetpool will
   default to 8 and 32, respectively, as defined in Neutron. For IPv6, the
   min_prefixlen and max_prefixlen of the subnetpool will default to 64 and
   ? respectively, as defined in Neutron).

#. A l3_policy is created with an ip_pool, ip_version, subnet_prefix_length,
   and address_scope, but no subnetpools are provided. In this case the
   ip_version of the l3_policy is ignored, and the subnetpool is created in
   this address_scope as before. If both, v4 and v6, address_scopes are
   provided, a subnetpool will be created for each family. If a subnetpool
   already exists in this address_scope, and overlaps with the CIDR of the
   ip_pool, an exception will be raised.

#. A l3_policy is created with an address_scope, and subnetpool(s). Since the
   ip_version is already defined for the address_scope and the CIDR and prefix
   lengths are defined for the subnetpool(s), the ip_version, ip_pool and
   subnet_prefix_length of the l3_policy will be ignored if provided.

#. A l3_policy is created with one or more subnetpools. The address_scope
   associated with the subnetpool(s) is associated with the l3_policy. If
   no address_scope is associated with the subnetpool(s), an exception will
   be raised. Similarly, it would not be valid to change the address_scope
   (or unset it) for a subnetpool associated with a l3_policy. An enforcement
   mechanism, most likely in the form of a Neutron mechanism driver that
   prevents such invalid configuration, will be needed.

In all of the above cases, if subnetpools are associated with an address_scope
that is associated with a l3_policy, they will be considered for subnet
allocation only if they are explicitly associated with the l3_policy.

A minor variation of all the above cases is the one where either or all of
ip_pool, ip_version, and subnet_prefix_length are not provided. In such cases
the GBP defaults for these attributes will be used (but overridden by the
corresponding attributes in the address_scope and subnetpools if any of those
are explicitly provided).

In the cases where address_scope is being explicitly provided (either directly
or via its association to subnetpools), the shared attribute of the
address_scope will be validated for consistency with the shared attribute of
the l3_policy during the creation of the l3_policy. If they are inconsistent,
an exception will be raised, and l3_policy creation will fail. If the shared
attribute of the l3_policy is subsequently updated, the shared attribute of the
address_scope and the subnetpool will be updated if the address_scope and
subnetpool was implicitly created by GBP.

It should be noted that the ip_pool, subnet_prefix_length, and ip_version
attributes of the l3_policy may only have effect at the l3_policy creation
time. In the body of response for GET l3_policy call, the ip_pool will be set
to a comma separated string consisting of a list of CIDRs corresponding to each
subnetpool currently present in the address_scope. To preserve API backward
compatibility (i.e. to not immediately break existing clients and integration
tests), the subnet_prefix_length and ip_version will be set to the
corresponding subnet_prefix_length and ip_version when only one CIDR is
present. If more than one CIDR is present, these will be set to null. In the
case where multiple CIDRs are present, more details like the prefix length of
the subnets that are drawn from the subnetpools corresponding to these CIDRs
can be obtained by navigating the resource relationship from l3_policy to
address_scope to subnetpool.


Data model impact
-----------------

The mapping for the l3_policy will be modeled in the DB as follows:

::

 class L3PolicySubnetpoolAssociation(model_base.BASEV2):
     """Models the 1 to many relation between L3Policies and Subnetpools."""
     __tablename__ = 'gp_l3_policy_subnetpool_associations'
     l3_policy_id = sa.Column(sa.String(36), sa.ForeignKey('gp_l3_policies.id'),
                              primary_key=True)
     subnetpool_id = sa.Column(sa.String(36), sa.ForeignKey('subnetpools.id'),
                               primary_key=True)


 class L3PolicyMapping(gpdb.L3Policy):
     """Mapping of L3Policy to set of Neutron Routers."""
     __table_args__ = {'extend_existing': True}
     __mapper_args__ = {'polymorphic_identity': 'mapping'}
     address_scope_v4_id = sa.Column(
         sa.String(36), sa.ForeignKey('address_scopes.id'), unique=True)
     address_scope_v6_id = sa.Column(
         sa.String(36), sa.ForeignKey('address_scopes.id'), unique=True)
     subnetpools = orm.relationship(L3PolicySubnetpoolAssociation,
                                    cascade='all', lazy="joined")
     routers = orm.relationship(L3PolicyRouterAssociation,
                                cascade='all', lazy="joined")

Note that address_scope_id and subnetpools are not nullable. It is either
created implicitly or provided explicitly, but is always required. This is a
backward incompatible DB change and a script will be provided to migrate data
from existing deployments to this new structure. The script will essentially
create an address_scope and subnetpool for each existing l3_policy.

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

This is how the udpated l3_Policy mapping would look like in terms of the mapping
extension definition

::

    gp.L3_POLICIES: {
        'address_scope_v4_id': {'allow_post': True, 'allow_put': False,
                                'validate': {'type:uuid_or_none': None},
                                'is_visible': True, 'default': None},
        'address_scope_v6_id': {'allow_post': True, 'allow_put': False,
                                'validate': {'type:uuid_or_none': None},
                                'is_visible': True, 'default': None},
        'subnetpools_v4': {'allow_post': True, 'allow_put': True,
                           'validate': {'type:uuid_list': None},
                           'is_visible': True, 'default': None},
        'subnetpools_v6': {'allow_post': True, 'allow_put': True,
                           'validate': {'type:uuid_list': None},
                           'is_visible': True, 'default': None},
        'routers': {'allow_post': True, 'allow_put': True,
                    'validate': {'type:uuid_list': None},
                    'convert_to': attr.convert_none_to_empty_list,
                    'is_visible': True, 'default': None},
        'ip_version': {'allow_post': True, 'allow_put': False,
                       'convert_to': conv.convert_to_int,
                       'validate': {'type:values': [4, 6, 46, 64]},
                       'default': 4, 'is_visible': True},
    },


Security impact
---------------

None


Notifications impact
--------------------

None


Other end user impact
---------------------

The l3_policy creation workflow has optional address_scope and subnetpools
arguments.This new workflow will be reflected in all clients and UI.


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
.. [#] https://review.openstack.org/#/c/419315/
