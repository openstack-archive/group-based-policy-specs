..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
Port Security Support in GBP Policy Targets
============================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/port-security-support-in-policy-target


Problem description
===================

Current GBP model applies security groups on all Policy Targets by default.
While this is fine for normal VMs, this will not work for advanced services
(eg. Firewall or Router). These VMs will require that the antispoofing
security rules to be not applied on these Service VM Policy Targets.


Proposed change
===============

Add a new api attribute named port_security_enabled to Policy Target resource.
The default value for port_security_enabled will be True in which case the PTs
will be associated with gbp security groups. If a Policy Target is created with
port_security_enabled attribute set as False, then the Neutron port that maps
to the Policy Target will be created with the port_security_enabled attribute
set as False for the Neutron port. With port_security_enabled attribute set
as False, Neutron requires that we do not pass security groups in the Neutron
port create/update request.


Data model impact
-----------------

Policy Target
  * ADD attribute port_security_enabled

The default value for port_security_enabled will be True if not specified.

A DB migration will be introduced to add a column named port_security_enabled
with a default value True to the table gp_policy_targets.

REST API impact
---------------

The REST API changes look like follows:

The following is added to Policy Target::

 POLICY_TARGETS: {
      'port_security_enabled': {'allow_post': True, 'allow_put': True,
      							'convert_to': attr.convert_to_boolean,
                                'default': True, 'enforce_policy': True,
                                'is_visible': True},
  }

Security impact
---------------

When a Policy Target is created with port_security_enabled set as False,
there will not be any iptable rules applied by the Neutron L2 agent. Even
the antispoofing rules will not be applied on the Neutron ports
that have port_security_enabled as False.

Setting Port Security as False is quite dangerous as it will open up the
network for spoofing. This operation will be limited to an admin only operation
and this is intended to be used only for Service VMs.


Notifications impact
--------------------


Other end user impact
---------------------


Performance impact
------------------


Other deployer impact
---------------------

Port security extension has to be enabled in the Neutron core plugin for this
feature to be used.


Developer impact
----------------


Alternatives
------------


Implementation
==============

Assignee(s)
-----------

Magesh GV (magesh-gv)


Work items
----------


Dependencies
============

This feature requires the Neutron Core plugin to have the port security
extension enabled.


Testing
=======


Documentation impact
====================


References
==========

https://blueprints.launchpad.net/neutron/+spec/ml2-ovs-portsecurity
