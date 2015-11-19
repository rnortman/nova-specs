..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================
 Scoped Policies for Server Groups
===================================

https://blueprints.launchpad.net/nova/+spec/server-group-scoped-policies

Server groups are currently used to ensure that instances in the groups are
either grouped together on the same host (affinity) or spread across different
hosts (anti-affinity).  This affinity/anti-affinity concept could be broadly
useful for a number of use cases, but many of these use cases would require
(anti-)affinity with a scope other than the host.  This spec adds a general and
flexible scoping mechanism to server group policies which enables these use
cases.  Under this mechanism, "host" is just one of (potentially) many
available scopes; by making host the default scope if none is specified, this
mechanism is fully backward-compatible with the existing server group policies.
Arbitrary additional scopes can be defined by the deployer.

Problem description
===================

Server group policies are currently limited to a single scope: a single host
machine.  As the use cases below illustrate, there are many use cases that
could be addressed by allowing different scopes.  Note that some of these use
cases could be addressed by the existing availability zones mechanism, but that
mechanism also has important limitations.

Use Cases
---------

* A tenant needs to allocate several instances for maximum redundancy.  To
  support this, the deployer wants to structure their infrastructure into
  maintenance zones and offer the assurance that to the extent possible, no two
  maintenance zones will be affected by maintenance at the same time.  The
  deployer, however, wants to ensure that capacity is used efficiently in the
  different zones, and therefore wants to avoid letting the end users explicity
  target particular maintenance zones (e.g., the current availability zone
  mechanism, which may result in unbalanced capacity utilization), instead
  letting the scheduler choose a zone based on current capacity in each zone,
  within the constraints set by the end user.

  - Using scoped server group policies, the deployer defines a maintenance zone
    scope, and the end user creates a server group with anti-affinity policy
    within this scope.  The scheduler will automatically distribute instances
    within the group across zones, while also taking into account available
    capacity in each zone, without requiring (or allowing) the end user to
    explicity target particular zones.

* An end user wants to build several instances that can share an IP address.
  The deployer's network is architected in such a way that IP addresses can
  only be shared within certain network segments.

  - The deployer defines an IP zone scope, and the end user creates a server
    group with affinity policy within this scope.

* An end user wants to build a database cluster where the instances forming the
  cluster must be on different hosts for redundancy, but should preferrably
  have minimum network latency and maximum bandwidth between them.

  - The deployer defines a "switch" scope which represents network segments
    with very fast network within each segment.  The end user creates a server
    group with a hard anti-affinity policy in the host scope, and a soft
    affinity policy within the switch scope.  This ensures that the instances
    will definitely be on different hosts (or the build will fail), and
    requests that they be on the same switch if possible, but the build will
    not fail if this is not possible.

* An end user builds a single instance to serve as a web front-end, assuming
  that only a single instance will be necessary (or otherwise forgetting to
  plan ahead).  The user now needs to scale and wants redundancy (as above).

  - The end user creates a server group with maintenance zone anti-affinity and
    adds the existing instance to it.  (Note that this does not cause the
    existing instance to move, but simply marks it as part of the group.)  The
    user can then build additional instances within the group with the
    assurance that no two will be in the same maintenance zone.

* An end user wants to build a large number of instances and ensure that
  they're roughly evenly distributed across maintenance zones (as above) for
  redundancy.  However, the number of instances is much larger than the number
  of maintenance zones available.

  - The user creates a server group with a soft anti-affinity policy in the
    maintenance zone scope.  The scheduler then attempts to spread instances
    evenly across zones, to the extent allowed by capacity, but allows more
    than one instance per zone.  (Note that deleting or removing instances from
    the group does not result in immediate rebalancing; only scheduling of new
    instances is affected, at which time the scheduler will attempt to restore
    balance to the extent allowed by available capacity.)

* An end user wants to target builds to a particular maintainence zone.  The
  available zones are defined by the deployer, but the deployer has an interest
  in obfuscating the zone identifiers so that each tenant identifies the zones
  through an identifier specific to the tenant -- in this way the deployer can
  reduce the chances of particular zones being more popular than others based
  on, for example, sort order of the identifiers.

  - The deployer defines the maintenance zone scope as explicitly targetable
    but with obfuscated IDs.  The true identifier will be hashed with the
    tenant ID and a salt in order to create surrogate IDs specific to each
    tenant.  These IDs can be used in server group policies for targeting.  All
    public APIs will accept and return these surrogate IDs.

Proposed change
===============

This change would allow deployers/operators to define arbitrary scopes for
server group policies and group hosts according to those scopes.  The host
aggregate mechanism is used to assign hosts to their scoped groups.  The
relationship between scopes and aggregates is one-to-many, while the
relationship between scopes and hosts is many-to-many.  A host can be in at
most one aggregate in each scope, but can be in more than one aggregate
associated with different scopes.  This allows scopes to be overlapping in
arbitrary ways, but within a single scope, each host can only be in one
aggregate.  (This cannot be enforced with database constraints using the data
model proposed here, but see the Alternatives section below for an approach
that would allow DB-level enforcement of this constraint.)

One scope is defined implicitly, and that is the host scope.  Each host is
implicitly the sole member of a virtual aggregate associated with this scope.
This enables backward compatibility with existing server group policies.

Adding/Removing Instances to/from Server Groups
-----------------------------------------------

To make this mechanism cover all the use cases above, we also need to add a
mechanism for adding/removing instances from a server group.  Currently the
server group can only be specified at boot time (via scheduler hint) and can
never be removed.  The boot-time mechanism will be preserved, but we will also
allow existing instances to be added/removed via a separate API call.  However,
to avoid race issues, we will not enforce policies during add-instance
operations; adding a host may result in policy violations.  Policies are
consulted and enforced only during scheduling (including scheduling during
initial boot and also during migrations).

Obfuscation of Identifiers
--------------------------

If a policy scope uses obfuscated identifiers, they shall be generated by a
Version 5 UUID as described in RFC 4122, in two passes.  (In python, this
method of UUID generation is implemented in the uuid.uuid5() standard library
function.)  The first pass generates a tenant-specific namespace UUID by using
a UUID defined when the scope is created (or assigned randomly at that time if
not specified) as the namespace paramter, and is the tenant ID converted to a
string without padding (e.g., "12345") as the name parameter.  The second pass
then uses this tenant-specific namespace UUID as the namespace parameter, and
the host aggregate identifier converted to a string (e.g., "67890") as the name
parameter.  For example::

  tenant_namespace_uuid = uuid.uuid5(policy_namespace_uuid, tenant_id)
  obfuscated_id = uuid.uuid5(tenant_namespace_uuid, aggregate_id)

Alternatives
------------

The existing server group policies and availability zones both address some,
but not all, of the use cases above.  In fact, both can be seen as special
cases of general scoped policies on server groups.

Of the two, availability zones address the largest subset of the use cases.
The primary limitation of availability zones is that the deployer must expose
the list of zones to the end user, and the end user then has both the
responsibility and power to explicitly target zones.  This can be inconvenient
for the end user, and can also create a serious resource allocation problem for
the deployer, because if end users can select zones explicitly, they will in
general tend to allocate instances unevenly between those zones.  For example,
if they can choose from zones 1, 2, and 3, then most end users will generally
allocate first to 1, then to 2, then to 3, resulting in many more allocations
to zone 1 than to zone 3.

The latter concern is addressed in this spec by allowing deployers to disable
explicit targeting, instead allowing tenants to set policies and allow the
scheduler to use capacity efficiently to satisfy the policy.  As an
alternative, for some use cases, the deployer can allow explicit targeting but
obfuscate to prevent the "zone 1" dog-piling effect.  (The configuration
options on the policy scope are allow_identifiers and obfuscate_identifiers;
both are boolean.)

The data model proposed here re-uses existing database tables to the extent
possible, at the expense of being able to enforce in the database the
requirement that each host be in at most one aggregate for each policy scope.
This requirement must therefore be enforced in the python code, which leads to
potential race conditions.  The scheduler, and any other code which makes use
of the host-to-scope mapping, will need to be written to be tolerant of
violations created by races.  (And by tolerant, that means returning an error
rather than blowing up or doing something ridiculous like blithely proceeding.)
There should also then be an audit proces to detect violations and alert
operators to manually fix the violations.

We could avoid all of that by enforcing the constraint in the database.  This
cannot be done using the existing host aggregate tables, and would require a
new database table or perhaps even two
(e.g. instance_group_policy_scope_host_groups and
instance_group_policy_scope_hosts; though strictly speaking only the latter is
required and would be the one with the important constraint).  This would need
to reside in the API database.  This may be worth it to avoid the headaches of
dealing with race conditions.


Data model impact
-----------------

A new InstanceGroupPolicyScope object and corresponding table will need to be
created to allow scopes to be created dynamically.  The object currently used
for the existing server groups (InstanceGroup) can be used for this spec as
well, but the semantics of the "policies" field with respect to server groups
will need to change in order to add scopes.  Currently these are stored as
strings with values "affinity" or "anti-affinity".  For scoped policies, we
will add an optional ":<scope>" suffix to that string (e.g., "affinity:zone");
if not specified, scope defaults to "host".  Additionally, a scope that allows
explicit group identification can have another ":<identifier>" suffix, so that
the full syntax would be, e.g., "affinity:zone:zone_A".  Identifiers in the
database are always the real internal identifiers, not the obfuscated
identifiers.

The database schema will add an instance_group_policy_scopes table and add a
column to instance_group_policies with a foreign key into the scopes table.

For mapping hosts to policy scopes, the existing host aggregate mechanism will
be used.  A metadata tag will map aggregates to policy scopes (just like the
current handling of availability zones).  This approach is subject to race
conditions; see discussion in Alternatives above.  (Note that existing
availability zones implementation is already just as subject to races.)

REST API impact
---------------

POST: v2.1/{tenant-id}/os-server-groups

  The value of the policies request parameter will be changed to an array of
  strings of the form described in the data model section above.  The formal
  syntax would be::

    "TYPE [ ":" SCOPE [ ":" IDENTIFIER ] ]",

  where TYPE is one of the enumerated values::

    ["anti-affinity", "affinity", "soft-anti-affinity", "soft-affinity"],

  SCOPE is a scope name defaulting to "host", and IDENTIFIER is a group
  identifier defaulting to null.  If the scope uses obfuscated identifiers,
  then it is always the tenant-specific obfuscated identifier which is used in
  the API.

  For example the following POST request body will be valid::

    {"server_group": {
        "name": "test",
        "policies": [
            "affinity:zone:acc173ed-867b-4e74-a69d-5586a490947b"]}}

  And will be answered with the following response body::

    {"server_group": {
        "id": "5bbcc3c4-1da2-4437-a48a-66f15b1b13f9",
        "name": "test",
        "policies": [
            "affinity:zone:acc173ed-867b-4e74-a69d-5586a490947b"
        ],
        "members": [],
        "metadata": {}}}

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}

  Update server group name and/or policies.  Changing policies may create
  policy violations; no error will be generated.  Example request body::

    {"server_group": {
        "name": "renamed test",
        "policies": [
            "affinity:zone:acc173ed-867b-4e74-a69d-5586a490947b",
            "anti-affinity:host"
        ]}}

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}/action

  Add an instance to the group (specify "add_instance" action).  Note that this
  does not enforce policies during the add; the operation will succeed even if
  it creates a policy violation.  See discussion above.  Example request::

    {"add_instance": {
        "instance_id": "9c975e5f-d7fe-427f-928c-c830ff644a0c"
        }}

POST: v2.1/{tenant-id}/os-server-groups/{server_group_id}/action

  Remove an instance from the group (specify "remove_instance" action).

GET: v2.1/{tenant-id}/os-server-groups/{server_group_id}/audit

  Audit policy compliance (can be used to find violations due to add/remove
  operations; violations can in principle be fixed via migration, which but
  there is no automated triggering of migrations).

  The response to this request identifies which host aggregate each member
  instance is in, for each scope that has a policy assigned for the group (and
  only for those policies; in the example below this is "zone" and "host").  If
  the policy scope is configured to allow identifiers, then these identifiers
  will be the same as used for other policy operations (i.e., obfuscated or not
  according to policy scope configuration).  If the scope is not identifiable,
  then the identifiers returned here will be ephemeral random surrogate
  identifiers which will change from one invocation of this API to another, but
  within a given response body, the identifiers will be consistent (i.e., two
  instances show the identifier if and only if they are in the same aggregate
  for that scope).

  Note that this API does not actually highlight policy violations in any way;
  finding violations is up to the caller.

  Example response::

   {"server_group_policy_audit": {
        "server_group_id": "5bbcc3c4-1da2-4437-a48a-66f15b1b13f9",
        "members": [
            {
                "instance_id": "990f1e6a-ced2-4d75-9fac-a8431a48d21e",
                "placements": {
                    "zone": "acc173ed-867b-4e74-a69d-5586a490947b",
                    "host": "02f7d1b8-a5a6-41ed-8829-c479861dcb44"
                    }},
            {
                "instance_id": "9c975e5f-d7fe-427f-928c-c830ff644a0c",
                "placements": {
                    "zone": "86db6549-e973-4eae-a13e-801a6ddeb880",
                    "host": "c74d2f27-6768-4418-b162-190942c0cfaf"
                    }}]}}
`
GET: v2.1/{tenant-id}/os-policy-scopes

  List policy scopes.

POST: v2.1/{admin-tenant-id}/os-policy-scopes

  Create policy scope (admin-only)::

    {"policy_scope" : {
        "name": "maintenance_zone",
        "allow_identifiers": "true",
        "obfuscate_identifiers: "true"}}

  Response::

    {"policy_scope" : {
        "id": "6ab5b222-390b-450d-b6c9-e72e53ac1922",
        "name": "maintenance_zone",
        "allow_identifiers": "true",
        "obfuscate_identifiers: "true",
        "obfuscate_namespace_uuid: "6f72348f-df5d-4e0f-a043-4be92996dbfe"}}
        "aggregates": []}}

GET: v2.1/{tenant-id}/os-policy-scopes/{policy_scope_id}

  Show details for a policy scope, including group identifiers if the scope
  allows group identifiers to be used.  If identifiers are obfuscated, they
  will be obfuscated for normal users but the real identifiers will be shown to
  admin users.  Admin users can add an optional "as_tenant_id" query parameter
  to view obfuscated identifiers as a particular tenant would see them, under
  the field "obfuscated_group_id".

POST: v2.1/{admin-tenant-id}/os-policy-scopes/{policy_scope_id}

  Update a policy scope (admin-only)

DELETE: v2.1/{admin-tenant-id}/os-policy-scopes/{policy_scope_id}

  Delete a policy scope (admin-only)

POST: v2.1/{admin-tenant-id}/os-policy-scopes/{policy_scope_id}/action

  Add an aggregate to the scope (admin-only; specify "add_aggregate" action)

POST: v2.1/{admin-tenant-id}/os-policy-scopes/{policy_scope_id}/action

  Remove an aggregate from the scope (admin-only; specify "remove_aggregate"
  action)



The above API changes will be introduced in a new API microversion.

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

Python-novaclient, and any other API front-ends, will need to be modified to
take advantage of the new capabilities, but all public APIs are
backward-compatible, so clients can also continue to operate without
modification, but will not be able to access the new features.

Performance Impact
------------------

Performance will only be affected during scheduling.  The scheduler will need
to take into account quite a bit of information whenever scheduling an instance
that belongs to a server group, namely the hosts of all the other instances in
the group, which will need to be mapped to the host aggregates associated with
the scope(s) for the server group's policies.  It may be worth allowing a
deployer-specied maximum or quota on the number of members of a given group to
limit the potential impact.

Other deployer impact
---------------------

These changes will be fully backward-compatible, so that if deployers do
nothing they get the same behavior that they currently have.  If deployers want
to take advantage of these features, they'll need to manually define policy
scopes, create host aggregates for the scopes, assign hosts to these
aggregates, and (optionally) set policies to limit what end users can do with
scopes.

Developer impact
----------------

Some objects and internal APIs will change.

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

- Implement data model changes

- Implement config options

- Implement scheduler changes

- Implement APIs

- Update python-novaclient

Dependencies
============

None.

Testing
=======

WIP

Documentation Impact
====================

WIP

References
==========


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
