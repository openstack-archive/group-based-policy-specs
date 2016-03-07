..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Status for GBP Resources
==========================================

Launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/inject-default-route


Problem description
===================

GBP supports a configurable Policy Driver based design. Rendering of GBP policy
model can be performed by the Policy Driver in an asynchronouse manner.
However, currently there is no mechanism to reflect the state of a particular
resource while it's being rendered and/or after its successful completion or
failure. This requirement has also come up in the context of the service chain
support [#]_.

.. [#] https://bugs.launchpad.net/group-based-policy/+bug/1479706


Proposed change
===============

Reflecting the operational state of a resource is typically achieved in
OpenStack projects by maintaining a ``status`` attribute [#]_[#]_. 

.. [#] https://github.com/openstack/neutron/blob/master/neutron/common/constants.py#L18-L26
.. [#] http://docs.openstack.org/developer/nova/vmstates.html

The status value will reflect one of the following states:

ACTIVE: The backend state of the resource is healthy and the user should expect
the configured state of the resource to be in effect

ERROR: The backend state of the resource is in error and the user should not
expect this resource to be operational

BUILD: The backend state of the resource is transient and may not be
operational but not in error state

The state transition diagram for the above states is as follows:

::

               +-----------+
    +----------+           +--------+
    |          |  BUILD    |        |
    |     +---->           <--+     |
    |     |    +-----------+  |     |
    |     |                   |     |
    |     |                   |     |
 +--v-----+---+          +----+-----v-+
 |            +---------->            |
 |  ERROR     |          |  ACTIVE    |
 |            <----------+            |
 +------------+          +------------+

In addition to the status attribute it is proposed to also add a 
``status_details`` attribute. This will be a free form string of a reasonable
length that provides more granular and likely more backend-specific information
about the status.

Both ``status`` and ``status_details`` attributes will be read-only attributes
in the API and updated only by the internal implementation.

For backward compatibility, we will allow these two attributes to be set to None. When
set to None it implies that the backend implementation does not support the
``status`` attribute.


Data model impact
-----------------

import sqlalchemy as sa

sa.Column('status', sa.Enum('active', 'build', 'error',
          name='resource_status'), nullable=True)

sa.Column('status_details', sa.String(length=4096), nullable=True)


REST API impact
---------------


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

None anticipated.


Other deployer impact
---------------------


Developer impact
----------------


Community impact
----------------


Alternatives
------------


Implementation
==============

Assignee(s)
-----------


Work items
----------


Dependencies
============


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


Developer Documentation
-----------------------


References
==========


