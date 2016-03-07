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

For backward compatibility, we will allow these two attributes to be set to
empty string. When set to None it implies that the backend implementation does
not support the ``status`` attribute.


Data model impact
-----------------

import sqlalchemy as sa

sa.Column('status', sa.Enum('active', 'build', 'error',
          name='resource_status'), nullable=True)

sa.Column('status_details', sa.String(length=4096), nullable=True)


REST API impact
---------------

The following update will be made to each resource's attribute map definition:

::

        'status': {'allow_post': False, 'allow_put': False,
                   'validate': {'type:string': None},
                   'is_visible': True, 'default': ''},
        'status_details': {'allow_post': False, 'allow_put': False,
                           'validate': {'type:string': None},
                           'is_visible': True, 'default': ''},

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

Mitaka GBP client will be needed to read resource status.

Developer impact
----------------

Policy driver implementation should appropriately set the status for GBP
resources.

Community impact
----------------

Helps to achieve asynchronous behavior with GBP API.


Alternatives
------------

None


Implementation
==============

The intial patch will only update the GBP resource and DB model. The
setting of resource status will be implemented in the planned asynchronous
policy driver [#]_.

.. [#] https://blueprints.launchpad.net/group-based-policy/+spec/async-policy-driver

Client will be updated to report status attributes. Updates to UI and Heat will be
performed as follow up patches.

Assignee(s)
-----------

snaiksat


Work items
----------

API and DB layer updates.


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

None


API Tests
---------

UTs


Documentation impact
====================

User Documentation
------------------

Will provided with the new async policy driver.


Developer Documentation
-----------------------

Devref document will be added.

References
==========


