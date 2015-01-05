..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================================
Modifying additional IPs to support sharing IPs in Neutron
==========================================================

https://blueprints.launchpad.net/neutron/+spec/allow-sharing-additional-ips

This spec is meant to lay out a plan to modify the existing additional IP
feature to allow us to share those IPs to other instances. A shared IPs feature
is intended to provide a solution in Neutron for users who wish to have a
high-availability active-passive set up between multiple instances using the
same IP. This would remove the requirement to leverage other, not as viable,
ways of solving the same use case. A typical implementation of this would be
two servers maintaining a heartbeat watch on each other and taking ownership
(via network configuration and gratuitous ARP) of the shared IP to divert
traffic to themselves when it recognizes that the respective peer node has
failed. 

Problem Description
===================

Neutron currently lacks a way to share an IP between multiple ports to support
HA active-passive application configurations. Some solutions, like load
balancers, solve active-active configurations, but this requires additional
resources and also creates a single-point-of-failure scenario, so HA still
isn't solved. A strikingly similar solution for the HA active-passive problem
is already supported by other cloud providers currently[1].

Proposed Change
===============

Introduce an 'enabled' flag that will be set to True for the first v4 and v6 IP
on a Port, and False for all additional IPs afterwards. This flag will then be
handled by Nova, and any other dependent services, to omit network
configurations for disabled IPs on the guest. This will then allow any
additional IP to be shared between Ports and therefore instances. This way, the
user may configure whichever instance they want to take ownership of the IP
without the risk of the other trying to hijack the IP on restart, or any
command that would cause the undesired guest to try and take ownership of the
shared IP without the user orchestrating it. We would also introduce a
read-only 'shareable' attribute for the fixed_ips elements to indicate which
IPs on the port can be shared to others within the same network segment.

The following should therefore hold true once implemented:

#. IPs that are enabled CANNOT also be shared

#. Conversely, IPs that are shareable CANNOT also be enabled

#. The "shareable" attribute CANNOT change while the IP is allocated

#. The first v4 and/or v6 on a Port is considered implicitly the Primary IP,
   all others are secondary. This should hold true for both port create and
   update.

#. IPs that are enabled should not be configured guest-side automatically.
   Instead, the user should configure these manually; we should document how to
   do this appropriately and, ideally, provide tools to help users accomplish
   this task.

#. Any host-side orchestration should still occur to at least allow network
   connectivity once a guest takes ownership of the IP.

#. If a port has all of its IPs deallocated, when it receives new IPs, it will
   hold true that the first not-shared IP will have enabled set to True for
   each v4 and v6.

Data Model Impact
-----------------

One new column will need to be added: an "enabled" column that indicates
whether or not the guest should configure the networking information for the IP
on the Port in question.

We would need a "catch up" migration to set the appropriate data in the new
column. This could exist as part of the patch adding the new column, or as a
separate migration that lives with the implementation of shared IPs.

REST API Impact
---------------

The 'shareable' attribute will be added to the fixed_ip resource (under
fixed_ips list) for the Port resource. This will allow the user to know which
IPs are allowed to be shared between ports going forward.

+----------+-------+---------+---------+------------+--------------+
|Attribute |Type   |Access   |Default  |Validation/ |Description   |
|Name      |       |         |Value    |Conversion  |              |
+==========+=======+=========+=========+============+==============+
|shareable |bool   |RO       |         |            |indicates     |
|          |       |         |         |            |whether IP is |
|          |       |         |         |            |shareable     |
+----------+-------+---------+---------+------------+--------------+
|enabled   |bool   |RO       |         |            |indicates     |
|          |       |         |         |            |whether IP is |
|          |       |         |         |            |to be         |
|          |       |         |         |            |configured    |
+----------+-------+---------+---------+------------+--------------+

Security Impact
---------------

N/A

Notifications Impact
--------------------

N/A

Other End User Impact
---------------------

Once allocated, the use of an additional IP is up to the end user. If they
allocate one but don't configure their instances to use it, they'll potentially
be paying for a useless resource. Aside from that, there are no other foreseen
impacts. We should provide tooling and documentation to help.

Performance Impact
------------------

There is no additional performance overhead foreseen.

IPv6 Impact
-----------

This change should include the allowance of shared IPv6 addresses.

Other Deployer Impact
---------------------

The feature is contained entirely in the code and shouldn't require any
additional configuration or infrastructure.

Developer Impact
----------------

To fully support this feature, we would need an analogous change in Nova or any
other service/application that intends to consume Neutron. if the 'enabled'
field is False, Nova, et al, should explicitly tell the guest NOT to configure
the IP (through whatever means necessary). In Nova, this is likely through
ommission of the IP configuration related to the shared IP in any derived
network configuration files, e.g. Debian's interfaces file.

Community Impact
----------------

Planning to discuss this in the next Neutron weekly meeting to gauge this.

Alternatives
------------

Floating IPs and/or load balancers can be used for similar configurations to
satisfy some use cases, but not all. Shared IPs specifically provide us with an
easy way to have an active-passive setup. There is also an existing
'allowed-address-pairs' extension[2] which has some short-comings relative to
this specification. This leaves a question of whether we can take the existing
extension and modify, or if this spec lays out the desired approach going
forward for Neutron.

Load balancers
~~~~~~~~~~~~~~

The main issue here is that witout a shared IP at some level in the stack, even
if you put your application behind a load balancer, there is a single point of
failure scenario.

Floating IPs
~~~~~~~~~~~~

Floating IPs solve a different problem and in order to share a floating IP
between multiple nodes, you'd have to either orchestrate updating the mapping
that supports floating IPs on failover, or you'd have to back the floating IP
with a shared internal IP, which would then require a shared IP solution
anyway.

Allowed Address Pairs extension
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The one thing not supported here is allocation. It appears that one has to
create a dummy port to allocate an address, and then set that address as
allowed on N other ports.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thomas-maddox

Other contributors:
  cerberus

Work Items
----------

#. Add an 'enabled' flag to the association between Port and IP. This should be
   reflected in the 'fixed_ips' constituents of the Port resource.

#. Add logic to set the first v4 and v6 for a Port as 'enabled=True', and then
   all others as 'enabled=False'.

#. Add a 'shareable' read-only attribute under 'fixed_ips' Ports resources to
   indicate which IP addresses are shareable to other ports. This is the
   inverse of whether that Port/IP combination is 'enabled'.

#. Add handling for the 'enabled' flag to Nova, so it won't inject network
   information for disabled IPs.

#. Implement quotas to allow an operator to limit the number of additional IPs
   per port

Dependencies
============

As above, this requires an appropriate change in Nova to fully support, and
thus will require a distinct blueprint.

Testing
=======

Tempest Tests
-------------

Tempest tests to evaluate handling of the 'enabled' flag between Neutron and
Nova would be advisable to ensure that host and guest configurations are
handled appropriately when using additional IPs.

Functional Tests
----------------

Test that 'enabled' is being set appropriately for various scenarios:

#. No additional IPs at create produces only enabled IPs

#. Additional IPs at create produces first IP as enabled and all others as
   disabled

#. Additional IPs in update produces disabled IPs

#. Sharing IP to other port in update shares the IP to the other port and
   maintains disabled status on both ports

#. Sharing IP to other port in create (after it's already on another Port)
   shares IP to new port and maintains disabled status on both ports

#. Sharing enabled IP to port in update fails

#. Sharing enabled IP to port in create fails

#. One v4 and one v6 produces two enabled IPs in port creates

#. Two v4 and one v6 produces two enabled (one of each v4 and v6) that are
   enabled and one additional v4 address, which should be disabled

#. Two v4 and two v6 produces two enabled (one of each v4 and v6) that are
   enabled and two additional of each (v4 and v6) that are disabled


API Tests
---------

Test that web API entry points result in 'enabled' and 'shareable' flags are
being set appropriately for various scenarios:


Documentation Impact
====================

User Documentation
------------------

Additional documentation must be provided to indicate how to use the feature.
We also believe that information regarding WHY this feature is useful should be
part of that, both so that others can be educated and so no one accidentally
begins configuring IPs for themselves they don't know how to use.


Feature desription and how-to need to be added here:
`http://docs.openstack.org/api/openstack-network/2.0/content/\
General_API_Information-d1e436.html`

Developer Documentation
-----------------------

An API documentation update would help other developers and consumers of
Neutron understand the contract for this feature.

API reference: http://developer.openstack.org/api-ref-networking-v2.html

References
==========

[1] AWS currently using shared IPs:
https://aws.amazon.com/articles/2127188135977316

[2] Allowed Address Pairs:
`http://docs.openstack.org/api/openstack-network/2.0/content/\
allowed_address_pair_ext.html`
