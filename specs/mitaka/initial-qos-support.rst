..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Initial QoS Support
==========================================


Problem description
===================
Group-Based Policy for OpenStack currently does not offer any way to set QoS
(Quality of Service) policies, either within or across Policy Target Groups.


Proposed change
===============
After further discussion, a change will be proposed. For now, the following
statements should be considered for an initial QoS support:

Neutron has a QoS API since Liberty [1] and allows maximum bandwidth rate and
burst bandwidth rate to be set.

It does so at the Neutron Port level.

It is also possible to set it to Neutron Networks, in which case the child
Ports will inherit that policy (unless it is overridden by applying the policy
directly to the Port.

So, QoS policies cannot be applied to specific classifications of traffic.

There are 2 major places to apply QoS in GBP: Within a PTG and Across PTGs.
GBP already has a Policy Action Type "QoS" expressed through the Horizon
dashboard [2] - this would be the the "Across PTGs" scenario - but no
underlying implementation on the server to back it up.

As part of an initial QoS support in GBP, it is achievable to have QoS within
PTGs, i.e. to configure QoS rules for Policy Targets, since they map back to 
Neutron ports.

However, to support QoS across PTGs, the classification set as part of the
providing or consuming PRS would essentially be ignored, so at this point only
the ANY classifier could be supported. Unfortunately, since the QoS policies
supported by Neutron apply at the Port level, it would have a non-trivial
meaning when set as an Action of a PRS. One of the possible outcomes could be
that a consuming PTG of the PRS whose action is QoS, would have all its PTs
be automatically configured to respect the QoS policy.

These are the main thoughts that I have about QoS for now, please comment.
I am not setting this as WIP because it needs reviews/discussion on Gerrit.


Data model impact
-----------------
After further discussion.


REST API impact
---------------
After further discussion.


Security impact
---------------
After further discussion.


Notifications impact
--------------------
After further discussion.


Other end user impact
---------------------
After further discussion.


Performance impact
------------------
After further discussion.


Other deployer impact
---------------------
After further discussion.


Developer impact
----------------
After further discussion.


Community impact
----------------
After further discussion.


Alternatives
------------
After further discussion.


Implementation
==============

Assignee(s)
-----------
igordcard


Work items
----------


Dependencies
============
Neutron QoS API from Liberty or Mitaka.

Testing
=======

Tempest Tests
-------------


Functional Tests
----------------


API Tests
---------


Documentation impact
====================

User Documentation
------------------
Documentation will be impacted to address how QoS policies can be applied.


Developer Documentation
-----------------------


References
==========
[1] https://specs.openstack.org/openstack/neutron-specs/specs/liberty/qos-api-extension.html
[2] http://git.openstack.org/cgit/openstack/group-based-policy-ui/tree/gbpui/panels/application_policy/forms.py

