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
and 6 for IPv6. The value 46 may now be used to enable dual-stack support.

For each address family (IPv4 and/or IPv6) indicated by an l3_policy, there
will be a 1-1 mapping between GBP l3_policy and a Neutron address_scope,
and a 1-many mapping between GBP l3_policy and Neutron subnetpools. The 1-many
mapping allows for use cases where the l3_policy (and address_scope) is shared
across tenants, while the subnetpool is created per tenant and not shared.

The ip_pool parameter now allows comma-delimited prefixes, as well just a
single prefix. CIDRs from different address families may be used together
in the same ip_pool parameter. The ip_pool parameter may also be None,
which is intended for use with the default subnetpool Neutron extension.

The default subnetpool Neutron extension is used to control the desired behavior
when an implicit workflow is used on L3 policies with mapping drivers [#]_.
Default subnetpools are only used when the ip_version requests a given address
family (IPv4 or IPv6) and the user hasn't explicitly provided any CIDRs for
that address family (e.g. via the ip_pool parameter or a subnetpool).

Within each indicated address family, the subnetpools that are explicitly
associated with the l3_policy must all belong to the same address_scope. GBP
will allocate subnets only from the subnetpools that are associated with the
corresponding l3_policy. The list of subnets associated with the l3_policy can
be updated by adding or removing subnetpools (subnetpools which are currently
being used cannot be removed), or by updating the prefixes attribute of
subnetpools already associated with the l3_policy

The definition of the subnet_prefix_length is modified to refine it's use.
The subnet_prefix_length now only applies to IPv4 prefixes. It also is
used as the default subnet prefix length for all subnets allocated from
IPv4 subnetpools for the given l3_policy. In order to help backward-
compatibilty, values longer than 30 will now be ignored. IPv6 subnets
that are allocated will always use a prefix length of 64.

Two properties of address scopes are address namespace and routability.
Address namespace simply means that addresses are guaranteed to be unique
within an address scope. Routability means that a neutron router may only
forward between interfaces that have subnets which were allocated from
subnetpools with the same address scope (note: forwarding is still possible
via NAT when they aren't in the same scope).  It's worth pointing out that
since address scopes can span multiple l3_policies, the address uniqueness is
applied across the l3_policies. However, even though sharing a common address
scope implies routability across l3_policies, actual connectivity between
l3_policies is not implied because each l3_policy maps to a distinct neutron
router.  In GBP, connectivity between l3_policies remains subject to a PTG
providing a PRS that is consumed by an external_policy and a PTG in the
other l3_policy consuming that external_policy.

Let's consider the various combinations that are possible based on the choice
of implicit and explicit arguments.

#. A l3_policy is created with an ip_pool, ip_version, and
   subnet_prefix_length but no address_scope or subnetpool(s) are provided. In
   this case an address_scope for each address family specified by ip_version
   is created, and a subnetpool with prefixes for each CIDR for that address
   family found in ip_pool is created in that address_scope. If the l3_policy
   is shared, the implcitily created address_scope and subnetpool will also be
   shared. The default_prefixlen for IPv4 subnetpools will be set to the
   subnet_prefix_length of the l3_policy, and for IPv6 subnetpools it will be
   set to 64 (the min_prefixlen and max_prefixlen of the IPv4 subnetpool will
   default to 8 and 32, and to 64 and 128 for the IPv6 subnetpool, as defined
   in Neutron).

#. A l3_policy is created with an ip_version and subnet_prefix_length
   (IPv4 only), but no address_scope, subnetpool(s), or prefix in ip_pool
   are provided for that address family. There are two possibilities in this
   scenario:

     *  A default subnetpool (defined in neutron) exists for an address family
        indicated by ip_version. In this case the default subnetpool and its
        address scope are associated with the l3_policy.

     *  A default subnetpool does not exist for an address family indicated
        by ip_version. In this case, an exception is raised.

#. A l3_policy is created with an address_scope of the indicated address
   family, but no subnetpools of that address family are provided. If there
   are CIDRs in ip_pool from the same address family, then they are used as
   prefixes to create a new subnetpool within the specified address scope.
   If a subnetpool already exists in this address_scope, and overlaps with
   the CIDR of the ip_pool, an exception will be raised. If no CIDRs from the
   same address family are found, a check is made to see if a default
   subnetpool sharing the same address scope exists. If it does, then that
   subnetpool is associated with the l3_policy. If none exists, then an
   exception is raised.

#. A l3_policy is created with an address_scope and subnetpool(s). Since the
   supplied subnetpool(s) define how subnets will be allocated, any supplied
   ip_pool for that address family is ignored. The ip_pool value returned
   from any l3_policy API call will contain the set of prefixes used for
   allocating subnets for all address families specified in ip_version. In
   this case, that will be the aggregate prefixes from the explicitly
   passed subnetpools.

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

A minor variation of all the above cases is the one where either or both of
the ip_pool and subnet_prefix_length are not provided. In such cases the GBP
defaults for these attributes will be used (but overridden by the
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
to a comma delimited string consisting of a list of CIDRs made up of the
prefixes attribute values of all subnetpools associated with the l3_policy.


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

Additional tables will be added to track the Neutron address_scope
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


The size of the ip_pool column in the L3Policy table is increased from
64 to 256 in order to account for more than a single prefix. Once all
policy drivers use subnetpools, this column could be removed from
the DB.

::

 class L3Policy(model_base.BASEV2, BaseSharedGbpResource):
     """Represents a L3 Policy with a non-overlapping IP address space."""
     __tablename__ = 'gp_l3_policies'
     type = sa.Column(sa.String(15))
     __mapper_args__ = {
         'polymorphic_on': type,
         'polymorphic_identity': 'base'
     }
     ip_version = sa.Column(sa.Integer, nullable=False)
     ip_pool = sa.Column(sa.String(256))
     subnet_prefix_length = sa.Column(sa.Integer)
     l2_policies = orm.relationship(L2Policy, backref='l3_policy')
     external_segments = orm.relationship(
         ESToL3PAssociation, backref='l3_policies',
         cascade='all, delete-orphan')

A similar increase (64 to 256) is needed to the proxy_ip_pool parameter in the
DB for that extension, as well as the ability to make the field nullable:

::

 class ProxyIPPoolMapping(model_base.BASEV2):
     __tablename__ = 'gp_proxy_ip_pool_mapping'

     l3_policy_id = sa.Column(
         sa.String(36), sa.ForeignKey('gp_l3_policies.id', ondelete="CASCADE"),
         primary_key=True)
     proxy_ip_pool = sa.Column(sa.String(256), nullable=True)
     proxy_subnet_prefix_length = sa.Column(sa.Integer, nullable=False)

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
    },

In addition, the l3_policy itself needs modifications to support:

    * the new value 46 for ip_version:
    * a type change for ip_pool (subnet to string)

the defaultt value for ip_pool is defined in a new config file
variable. The variable has the same value as the old default, which
ensures backwards-compatibility, but also allows for the default
value to be something else (e.g. dual-stack prefixes, empty prefix,
IPv6 prefix only, etc.).

::

    L3_POLICIES: {
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
        'ip_version': {'allow_post': True, 'allow_put': False,
                       'convert_to': conv.convert_to_int,
                       'validate': {'type:values': [4, 6, 46]},
                       'default': 4, 'is_visible': True},
        'ip_pool': {'allow_post': True, 'allow_put': False,
                    'validate': {'type:string_or_none': None},
                    'default': GBP_CONF.default_ip_pool, 'is_visible': True},
        'subnet_prefix_length': {'allow_post': True, 'allow_put': True,
                                 'convert_to': conv.convert_to_int,
                                 # This parameter only applies to ipv4
                                 # prefixes. For IPv4 legal values are
                                 # 2 to 30. For ipv6, this parameter
                                 # is ignored
                                 'default': 24, 'is_visible': True},
        'l2_policies': {'allow_post': False, 'allow_put': False,
                        'validate': {'type:uuid_list': None},
                        'convert_to': conv.convert_none_to_empty_list,
                        'default': None, 'is_visible': True},
        attr.SHARED: {'allow_post': True, 'allow_put': True,
                      'default': False, 'convert_to': conv.convert_to_boolean,
                      'is_visible': True, 'required_by_policy': True,
                      'enforce_policy': True},
        'external_segments': {
            'allow_post': True, 'allow_put': True, 'default': None,
            'validate': {'type:external_dict': None},
            'convert_to': conv.convert_none_to_empty_dict, 'is_visible': True},
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

.. [#] https://docs.openstack.org/developer/neutron/devref/address_scopes.html
.. [#] https://review.openstack.org/#/c/282021/
