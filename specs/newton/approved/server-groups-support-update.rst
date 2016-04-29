..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================================
 Support Update Operations on Server Groups
============================================

https://blueprints.launchpad.net/nova/+spec/server-groups-support-update

Right now it is not possible to change a server group's policies after the
group is created, nor is it possible to remove instances from a group, nor to
add an instance to a group after boot time.  This spec adds these operations
and clarifies the semantics of existing operations (like migrations) with
respect to server group policies.



Problem description
===================

The key difficulty with these update operations is that they can all result in
policy violations.  This spec proposes semantics for these operations that
allow them to create policy violations without raising errors, and suggests
that the semantics of policies are that they are enforced at scheduling time
only.  This spec adds an API that allows users to retrieve the data necessary
for auditing policies to find violations.

Not that even in the existing implementation, policy violations can be created
if an instance is migrated by the admin to a host which does not match the
policy.

Use Cases
---------

A user may create an instance without a server group and later decide to create
more related instances, and may want affinity or anti-affinity for those new
instances with the existing instance.  A user may also want to assemble a set
of existing servers into a group, find policy violations, and trigger
migrations to eliminate the violations.

Changing the policy of an existing server group is probably not useful at
present, but see this related spec for which it may be quite useful:

https://review.openstack.org/#/c/247654/

Proposed change
===============

This spec adds APIs to add/remove instance to/from a group, update the policy
of a group, and retrieve information necessary to audit present policy
compliance among instances in the group.

Alternatives
------------

The primary alternative is to not allow policy violations to occur by returning
error on any operation that creates a violation.  This would include migrations
to a forced host (which will currently succeed despite the violation).  The
primary problem with this approach -- aside from the fact that it changes the
existing behavior in a way that operators probably do not want -- is that it is
difficult to implement without race conditions.

Data model impact
-----------------

None.

REST API impact
---------------

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}

  Update server group name and/or policies.  Changing policies may create
  policy violations; no error will be generated.  Example request body::

    {"server_group": {
        "name": "renamed test",
        "policies": ["affinity"]
        }}

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}/action

  Add an instance to the group (specify "add_instance" action).  Note that this
  does not enforce policies during the add; the operation will succeed even if
  it creates a policy violation.  See discussion above.  Example request::

    {"add_instance": {
        "instance_id": "9c975e5f-d7fe-427f-928c-c830ff644a0c"
        }}

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}/action

  Remove an instance from the group (specify "remove_instance" action)::

    {"remove_instance": {
        "instance_id": "9c975e5f-d7fe-427f-928c-c830ff644a0c"
        }}

GET: v2.1/{tenant-id}/os-server-groups/{server_group_id}/audit

  Audit policy compliance (can be used to find violations due to add/remove
  operations).

  The response to this request identifies which host each member instances is
  in.  If the user is an admin, actual host identifiers will be returned.
  Otherwise, ephemeral random surrogate identifiers will be generated.  These
  will change from one invocation of this API to another, but within a given
  response body, the identifiers will be consistent (i.e., two instances show
  the same identifier if and only if they are on the same host).

  Note that this API does not actually highlight policy violations in any way;
  finding violations is up to the caller.

  Example response::

   {"server_group_policy_audit": {
        "server_group_id": "5bbcc3c4-1da2-4437-a48a-66f15b1b13f9",
        "members": [
            {
                "instance_id": "990f1e6a-ced2-4d75-9fac-a8431a48d21e",
                "placements": {
                    "host": "02f7d1b8-a5a6-41ed-8829-c479861dcb44"
                    }},
            {
                "instance_id": "9c975e5f-d7fe-427f-928c-c830ff644a0c",
                "placements": {
                    "host": "c74d2f27-6768-4418-b162-190942c0cfaf"
                    }}]}}


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

The audit API may generate a very large response if there are many members of
the group.  It may be worth putting a configurable upper limit on the number of
server group members.

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  r-nortman

Other contributors:
  None

Work Items
----------

TBD

Dependencies
============

None

Testing
=======

This change should not require any new tempest tests.  Standard unit and
functional tests will be added.

Documentation Impact
====================

The new APIs will be documented, and the semantics of server group policies
with respect to migrations will be clarified.


References
==========

The following related spec adds policy scopes to make server group policies
more useful:

https://review.openstack.org/#/c/247654/


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
