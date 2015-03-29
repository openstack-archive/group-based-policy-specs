..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Service Chain driver base class
===============================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/service-chain-base

This blueprint proposes and specifies a list of changes to be carried out in
order to add a base class for Service Chain drivers, from which the remaining
drivers will extend.


Problem description
===================

Service Chain drivers for the Group-based Policy project currently do not
extend from any base class, which in turn means that there is no formal
interface for interacting with Service Chain drivers.


Proposed change
===============

A base class for Service Chain drivers will be added, defining the interface
to interact with these drivers. The existing Service Chain drivers will be
changed to extend from this base class. Furthermore, the new class will also
define context interfaces for each of the Service Chain resource types. The
class meant for service chain context will be changed to conform to the new
interfaces as well.

Alternatives
------------

None

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

Discuss things that will affect other developers working on OpenStack,
such as:

* If the blueprint proposes a change to the API, discussion of how other
  plugins would implement the feature is required.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  igordcard (Igor Duarte Cardoso)

Work Items
----------

* Add base driver class and change existing
  drivers and context classes to conform


Dependencies
============

None


Testing
=======

None


Documentation Impact
====================

None


References
==========

.. [1] Service chain driver base class Edit - Bug Report,
   https://bugs.launchpad.net/group-based-policy/+bug/1387433