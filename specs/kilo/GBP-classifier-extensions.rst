..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Group Based Policy Extensions - Classifier
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/group-based-policy/+spec/gbp-classifier-extension

Group Based Policy [1] was introduced in the Juno release. However, it has
some limitations which are addressed in this blueprint. 



Problem description
===================
The following issues are described in this blueprint:

1. The GBP Policy Classifier only allows matching on TCP/UDP port ranges 
and protocol values: TCP, UDP, ICMP. There is a need to extend this to
other fields  within the packet including:

 1. L7 fields. These would include fields such as a URL.
 2. Protocol path. The protocol path is the list of protocols encapsulating
    a packet payload and allows matching on a specific set of encapsulating
    protocols. This path may vary depending on whether or not some form of
    tunneling is used, for example a protocol path may be 
    {ip, gre, ip, tcp, *}, or {ip, udp,*}, where * is a wildcard.

2. The GBP Policy classifier needs a negation attribute to allow for the
case where packets that do not match the classifier may result in an action.



Proposed change
===============
This work is intended to be done during the Kilo release.

A. Policy Classifier Extensions

The following fields are added to the GBP Policy Classifier:

 1. l7_match_fields
 2. protocol_path
 3. negate

The new attributes for the Policy Classifier resource are shown below.

+-------------+----------+--------+---------+----+-------------------+
|Attribute    |Type      |Access  |Default  |CRUD|Description        |
|Name         |          |        |Value    |    |                   |
+=============+==========+========+=========+====+===================+
|id           |string    |RO, all |generated|R   |identity           |
|             |(UUID)    |        |         |    |                   |
+-------------+----------+--------+---------+----+-------------------+
|tenant_id    |uuid-str  |RO, all |from auth|CR  |Tenant Id          |
|             |          |        |token    |    |                   |
+-------------+----------+--------+---------+----+-------------------+
|name         |string    |RW, all |''       |CRU |human-readable     |
|             |          |        |         |    |name               |
+-------------+----------+--------+---------+----+-------------------+
|description  |string    |RW, all |''       |CRU |human-readable     |
+-------------+----------+--------+---------+----+-------------------+
|port_range   |List(dict)|RW, all |None     |CRU |Ranges of ports    |
|             |          |        |         |    |[{'min': '80',     |
|             |          |        |         |    |'max': '82'}]      |
+-------------+----------+--------+---------+----+-------------------+
|protocol     |Enum      |RW, all |None     |CRU |'tcp','udp,'icmp'  |
+-------------+----------+--------+---------+----+-------------------+
|direction    |Enum      |RW, all |Bi       |CRU |'in','out','bi'    |
+-------------+----------+--------+---------+----+-------------------+
|negate       |Bool      |RW, all |false    |CRU |true, false        |
+-------------+----------+--------+---------+----+-------------------+
|l7_matches   |List(dict)|RW, all |None     |CRU |List of match      |
|             |          |        |         |    |fields, eg,        | 
|             |          |        |         |    |[{'type': 'url',   |  
|             |          |        |         |    |'value':           | 
|             |          |        |         |    |'www.google.com'}] | 
+-------------+----------+--------+---------+----+-------------------+
|protocol_path|List(Enum)|RW, all |None     |CRU |List of protocol   |
|             |          |        |         |    |Enums, eg,         | 
|             |          |        |         |    |['ip,'tcp','*']    | 
+-------------+----------+--------+---------+----+-------------------+



Alternatives
------------

There are no proposed alternatives to what is proposed in this document.


Data model impact
-----------------

Migration scripts will be added to support the new attributes.


REST API impact
---------------
Changes to group_policy.py are shown in the following sections.

New Policy Classifier attributes::

  RESOURCE_ATTRIBUTE_MAP = {

    POLICY_CLASSIFIERS: {
        'id': {'allow_post': False, 'allow_put': False,
               'validate': {'type:uuid': None},
               'is_visible': True, 'primary_key': True},
        'name': {'allow_post': True, 'allow_put': True,
                 'validate': {'type:string': None},
                 'default': '', 'is_visible': True},
        'description': {'allow_post': True, 'allow_put': True,
                        'validate': {'type:string': None},
                        'is_visible': True, 'default': ''},
        'tenant_id': {'allow_post': True, 'allow_put': False,
                      'validate': {'type:string': None},
                      'required_by_policy': True,
                      'is_visible': True},
        'protocol': {'allow_post': True, 'allow_put': True,
                     'is_visible': True, 'default': None,
                     'convert_to': convert_protocol,
                     'validate': {'type:values': gp_supported_protocols}},
        'port_range': {'allow_post': True, 'allow_put': True,
                       'validate': {'type:port_range': None},
                       'convert_to': convert_port_to_string,
                       'default': None, 'is_visible': True},
        'direction': {'allow_post': True, 'allow_put': True,
                      'validate': {'type:values': gp_supported_directions},
                      'default': None, 'is_visible': True},
        'match_fields': {'allow_post': True, 'allow_put': True,
                      'validate': {
		         'type:list_of_dict_or_none': {
		           'type': {'type:not_empty_string': None, 'required': True},
		           'value': {'type:not_empty_string_or_none': None, 'required': True}}}    
                      'default': None, 'is_visible': True},
        'protocol_path': {'allow_post': True, 'allow_put': True,
                     'is_visible': True, 'default': None,
                     'convert_to': convert_to_list,
                     'validate': {'type:enum_list': None},
        'precedence': {'allow_post': True, 'allow_put': True, 'default': 1,
                      'convert_to': attr.convert_to_int,
                      'is_visible': True},
    },




Security impact
---------------

Existing security for Group-Based Policy [1] applies.

Notifications impact
--------------------

No change from Group-Based Policy.

Other end user impact
---------------------

The CLI for creating a Policy Classifier::

       neutron gp-policy-classifier-create 
           -–l7-match-fields "url:www.google.com"
           -–protocol-path "ip:udp"
           -–precedence "2"


Changes will be made to python-neutronclient to support this CLI.
Changes will also be made to Horizon and Heat.

Performance Impact
------------------

No significant performance impact.


Other deployer impact
---------------------
No other impact.

Developer impact
----------------
None.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Louis Fourie (louis.fourie@huawei.com)

Other contributors:
 Cathy Zhang
 Nicolas Bouthors


Work Items
----------
The work items include:

 1. Implement CRUD operations to specify a new fields
 2. Implement Database updates to support that API
 3. Implement Migration scripts to support the new DB items.
 4. Implement python-neutronclient chnages for CLI
 5. Implement Horizon changes.


Dependencies
============

This blueprint depends on Group Based Policy [1].


Testing
=======

Unit tests will be provided.


Documentation Impact
====================
A description of the new attributes must be added to the documentation.


References
==========

[1] Group-based policy https://review.openstack.org/#/c/87825

