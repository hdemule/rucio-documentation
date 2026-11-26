---
id: role_based_access_control
title: Role-Based Access Control
sidebar_label: Role-Based Access Control
---

Role-based access control (RBAC) lets an instance grant accounts permissions on DIDs through roles.
Without RBAC, an account acts on the DIDs of the scopes it owns, and admins on every DID.
With RBAC, a role grants read, write or delete on the DIDs matching a pattern to every account holding it,
whatever scopes the account owns.

RBAC is disabled by default. While it is disabled, with the generic permission policy,
Rucio behaves as it does without RBAC: roles can be defined, but grant nothing.

:::warning
RBAC does not support multi-VO instances: roles and their permissions are shared by the whole instance,
whatever the VO of the accounts holding them.
:::

## Enabling RBAC

RBAC is enabled with the `enabled` option of the `rbac` section, either in `rucio.cfg`:

```cfg
[rbac]
enabled = True
```

or, if the option is not set in `rucio.cfg`, in the configuration table, which takes effect without restarting the servers:

```bash
rucio config set --section rbac --key enabled --value True
```

Before enabling RBAC on an existing instance, follow [Upgrading](#upgrading).

## Concepts

### Roles

A role has a name, an optional description and two flags:

- `assignable`: whether the role can be assigned to and removed from accounts.
  A role which is not assignable can only be assigned or removed with `force`,
  and the [synchronisation with an identity provider](#with-an-identity-provider) never assigns nor removes it.
- `locked`: whether the role is protected from changes. Changing the description,
  the `assignable` flag or the permissions of a locked role, or deleting it, requires `force`.
  New roles are locked by default. Unlocking a role needs no `force`.
  The [synchronisation with the policy package](#with-the-policy-package) never alters nor deletes a locked role.

A role still assigned to accounts can only be deleted with `force`, which also removes it from these accounts and drops its permissions.

The `admin` role is reserved. It is created with the database schema, not assignable, is always locked,
and can neither be deleted nor be granted permissions: only its `assignable` flag can be changed.
While RBAC is enabled, an account holding the `admin` role is an admin, as is an account with the `admin` account attribute.
The attribute keeps working, so enabling RBAC demotes no admin.

### Role assignments

A role is assigned to an account with an optional activation date (`activates_at`, now by default)
and an optional expiry date (`expires_at`, none by default).
An assignment grants the role only while it is active: from its activation date until its expiry date.
An expired assignment grants nothing, but stays in the database until it is removed,
e.g. by [cleaning the expired assignments](#rest-api).

### DID permissions

A DID permission grants a role operations on the DIDs matching a DID pattern.
A pattern is `SCOPE:NAME`, where each part may end with `*`, the only wildcard:

| Pattern     | Matches                                                   |
|-------------|-----------------------------------------------------------|
| `*:*`       | every DID                                                 |
| `data*:*`   | the DIDs of every scope starting with `data`              |
| `data:*`    | the DIDs of the scope `data`                              |
| `data:raw*` | the DIDs of the scope `data` whose name starts with `raw` |
| `data:raw1` | the DID `data:raw1`                                       |

A pattern without a name part stands for the whole scope: `data` is stored as `data:*`.
A wildcard scope part may only be followed by the `*` name part, so `data*:raw*` is refused.
Patterns are not checked against the existing scopes and DIDs: a DID created later is covered as soon as it exists.

There are three operations, each including the ones before it. On the matching DIDs:

- **read** (`r--`): seeing them in listings, reading their metadata, replicas and rules,
  adding replication rules on them (provided the account also reads all their content, see below) and attaching them to collections;
- **write** (`rw-`): read, and creating them (also as samples of a collection the account reads),
  attaching DIDs to them, detaching from them the DIDs the account owns, and managing the DIDs the account owns;
- **delete** (`rwd`): write, and setting and deleting the metadata of any of them, and detaching any of their content.

:::info
Write is non-destructive, except on the DIDs the account owns: it adds DIDs and content,
and only alters or removes the DIDs it owns. Delete is destructive on every DID the pattern matches, whoever owns it.
:::

Granting write stores `rw-`, and granting delete stores `rwd`. A permission may also grant no operation (`---`), see below.

The permissions of an account's roles are combined as follows:

- **Within a role, the most specific pattern decides.** For a given DID, only the most specific pattern of the role matching it counts.
  From the most to the least specific: a single DID (`data:raw1`), a name pattern (`data:raw*`, the longer the more specific),
  a whole scope (`data:*`) and a scope pattern (`data*:*`, the longer the more specific).
  For example, `rwd` on `data:*` and `r--` on `data:raw*` keep the raw DIDs read-only,
  and `---` on `data:secret*` withdraws the secret DIDs from `data:*` altogether.
- **Roles add up.** An account is granted what any of its roles grants. A role never restricts what another grants:
  `---` only withdraws DIDs from a less specific pattern of the same role.
- **There is no inheritance.** A permission applies to the DIDs its pattern matches, never to their content.
  Reading a dataset does not grant reading its files: listing its content only shows the files the account may read.

### How access is decided

With RBAC enabled, the generic permission policy lets an account perform an operation on a DID if:

1. it is root or an admin;
1. it owns the scope of the DID;
1. it owns the DID (it is the account of the DID), a role grants it write on the DID, and the operation is delete:
   ownership and write make delete, so that an account may alter and remove the DIDs it created.
   Owning a DID grants nothing else, and withdrawing write on a DID from its owner also withdraws delete;
1. a role grants it the operation on the DID.

Roles only add to what the generic policy already allows, except for the operations which now require to read the DIDs they involve, see below.

## What enabling RBAC changes

### Reading DIDs

With RBAC enabled, an account reads only the scopes it owns and what its roles grant: owning a DID does not let an account read it.
Root and admins read everything. Everything else is:

- filtered out of listings, e.g. of DIDs, content, parent DIDs, files, replicas, rules, locks, transfer requests and scopes.
  A scope is listed if the account reads some of its DIDs;
- refused with an access denied error (HTTP 403) when asked for directly, e.g. the metadata of a DID, its replicas,
  and so its download, or a rule on it.

A DID which does not exist is refused in the same way, unless the account could read a DID of that name,
so that the existence of a DID is not revealed.

A policy package can disable the filtering of some listings, for some roles or for every account, see
[Disabling listing filters](operator/policy_packages/policy_package_development.md#disabling-listing-filters).

:::note
The `/redirect` endpoint is not authenticated, so it cannot tell what the requester may read.
With RBAC enabled, it refuses every request.
:::

### Operations requiring to read DIDs

With RBAC enabled, the following operations require the account to read the DIDs they involve.
Before, any account could perform them on any DID:

- adding a replication rule of its own on a DID, including the rules given when creating a DID.
  A rule copies the whole content of a collection, so the account must also read all of it, recursively:
  every container, dataset and file the DID contains when the rule is created;
- adding a replica of an existing file on a user RSE;
- attaching DIDs to a collection, even to one in a scope the account owns;
- getting the transfer requests of a DID;
- declaring a replica suspicious, as the declaration tells whether the replica exists.
  A PFN which does not resolve to a DID is refused in the same way.

Root and admins are not affected, nor is an account owning the scope of the DID, or owning the DID with write on it.
Before enabling RBAC, grant accounts read, through roles, on the DIDs they need for these operations.

The following also apply with RBAC enabled:

- A role never grants a recursive metadata update, which rewrites the metadata of the whole content of a collection.
- A DID created through a role, in a scope the account does not own, cannot be given the metadata keys Rucio reserves.
- Only root, admins and the owner of the scope can create DIDs for another account.

### What read on a collection does not protect

:::warning
Read on a collection does not show its content, but does not keep its content from being read either.

- Read (`r--`) has no inheritance: reading a collection does not show its content in Rucio.
  The names, metadata, PFNs and downloads of its content stay filtered per DID.
- A replication rule copies the whole content of a collection, recursively, to any RSE where the account has quota.
  Once copied, the content can be read outside Rucio by anyone the RSE lets read it, e.g. through storage access or deterministic paths.
  So adding a rule on a collection requires reading all its content, not only the collection itself.
- That check is only done when the rule is created. What is attached to the collection afterwards is copied by the rule
  without being checked, e.g. a collection the account reads, attached to a collection it owns which already has a rule,
  brings its whole content along, even the DIDs the account may not read.
- To keep an account from reading a DID and its content, grant it no permission at all, not even `r--`,
  on the DID **and on every collection containing it**. To be safer, grant it nothing on the scope either, e.g. on `data:*`.
- RBAC hides and restricts data in Rucio, not on storage. Replicas already on an RSE are as exposed as without RBAC
  to anyone the storage lets read them. Data which must stay secret also needs storage permissions,
  e.g. path-scoped tokens or ACLs, on the RSEs holding it.
:::

## Managing roles

Only root and admins can manage roles, their permissions and their assignments.
Any account can list its own roles and the permissions of the roles it holds.

Roles can be managed while RBAC is disabled, e.g. to prepare them before enabling it.
Every response of the role endpoints then carries an `X-Rucio-Warning` header, as roles grant nothing until RBAC is enabled.

### Command line

Roles are managed with the `rucio role` commands.
The same operations are available in Python through the [`RoleClient`](pathname:///html/client_api/roleclient.html).

#### Roles

```bash
# List the roles, with --detail also their permissions
rucio role list --detail

# Add a role
rucio role add analysts --description "Reads the data scopes." --locked false

# Change its description, and lock it
rucio role update analysts --description "Reads the data scopes, writes the raw data." --locked true

# Delete it. --force also deletes a locked role, or one still assigned to accounts
rucio role remove analysts --force
```

A role added with `rucio role add` is assignable and **locked**, unless `--assignable false` or `--locked false` is given.
Unlocking a role and deleting one with `--force` ask for confirmation.

#### DID permissions

```bash
# Read on the scopes starting with data, read and write on the raw DIDs of data, nothing on its secret DIDs
rucio role permission did add analysts 'data*' r
rucio role permission did add analysts 'data:raw*' rw
rucio role permission did add analysts 'data:secret*' none

# List the permissions of the role
rucio role permission list analysts

# Remove a permission: its DIDs fall back to the next less specific pattern of the role
rucio role permission did remove analysts 'data:secret*'
```

The operations are given as letters, `r`, `w` and `d`, in any combination, or as `none` for `---`.
Adding a permission on a pattern replaces what the role had on it.
Quote the patterns containing `*`, so that the shell does not expand them.

Before adding or removing a permission whose scope part is a wildcard, e.g. `data*`,
the command shows the scopes it currently matches and asks for confirmation.
`rucio role permission list --detail` expands such patterns into the scopes they match.
The command also warns when a pattern withdraws operations granted by a less specific pattern of the role.

```bash
$ rucio role permission list analysts
Permissions associated with role 'analysts':
+------+--------------+--------------+
| TYPE | IDENTIFIER   | OPERATION(S) |
|------+--------------+--------------|
| did  | data*:*      | r--          |
| did  | data:raw*    | rw-          |
| did  | data:secret* | ---          |
+------+--------------+--------------+
```

Changing the permissions of a locked role requires `--force`.

#### Role assignments

```bash
# Assign the role, permanently to jdoe, and for December only to guest
rucio role account add analysts jdoe
rucio role account add analysts guest --activates-at 2026-12-01 --expires-at 2027-01-01

# Change the expiry date of an assignment, NEVER for none
rucio role account update analysts guest --expires-at NEVER

# List the accounts holding the role, and the roles of an account with their permissions
rucio role account list -r analysts
rucio role account list -a jdoe --detail

# Remove the role from an account
rucio role account remove analysts guest
```

Dates are given in UTC, as `YYYY-MM-DD`, `YYYY-MM-DDTHH:MM:SS` or `YYYY-MM-DD HH:MM:SS`.
Assigning or removing a role which is not assignable, such as the `admin` role, requires `--force`:

```bash
rucio role account add admin jdoe --force
```

### REST API

Roles are managed through the `roles` endpoint, enabled by default.
See the [REST API documentation](https://rucio.cern.ch/documentation/html/rest_api_doc.html#tag/Role) for the details of each route.

| Route                                     | Description                                                                                  |
|-------------------------------------------|----------------------------------------------------------------------------------------------|
| `GET /roles/status`                       | Tell whether RBAC is enabled.                                                                |
| `GET /roles/`                             | List the roles.                                                                              |
| `POST /roles/<role>`                      | Add a role, with its `description`, `assignable` and `locked`.                               |
| `PUT /roles/<role>`                       | Change the given `description`, `assignable` and `locked` of a role, with `force` if locked. |
| `DELETE /roles/<role>`                    | Delete a role, with `force` if it is locked or assigned.                                     |
| `GET /roles/<role>/permissions`           | List the permissions of a role.                                                              |
| `POST /roles/<role>/permissions/did`      | Grant `operations` on a `did_pattern`, replacing what the role had on it.                    |
| `DELETE /roles/<role>/permissions/did`    | Remove the permission of a role on a `did_pattern`.                                          |
| `GET /roles/<role>/accounts`              | List the accounts holding a role.                                                            |
| `POST /roles/<role>/accounts/<account>`   | Assign a role, with its optional `activates_at` and `expires_at`, and `force` if needed.     |
| `PUT /roles/<role>/accounts/<account>`    | Change the given dates of an assignment.                                                     |
| `DELETE /roles/<role>/accounts/<account>` | Remove a role from an account, with `force` if the role is not assignable.                   |
| `GET /roles/accounts/<account>`           | List the roles of an account, `@me` for the caller. With `detail`, also their permissions.   |
| `PUT /roles/accounts/<account>`           | Synchronise the roles of an account with an identity provider, see below.                    |
| `DELETE /roles/accounts/expired`          | Remove the expired assignments of every account. With `dry_run`, only report them.           |

A DID permission is given as its pattern and the list of its operations, possibly empty for `---`:

```json
{"did_pattern": "data:raw*", "operations": ["read", "write"]}
```

## Synchronising roles

### With the policy package

A policy package can define the roles of the instance, with their description, see
[Role module](operator/policy_packages/policy_package_development.md#role-module).
Synchronising the roles with the policy package, with `rucio.core.role.sync_roles_from_policy_package`:

- creates the roles the policy package defines which do not exist yet, assignable and unlocked;
- updates the description of the existing unlocked roles it defines;
- deletes the unlocked roles it does not define, including those created through the CLI or the REST API.

Locked and reserved roles are never touched. Permissions and assignments are managed in Rucio:
the synchronisation only removes those of the roles it deletes.
The synchronisation is applied in a single transaction: either all of it is applied, or none of it.
`rucio.core.role.sync_roles_from_policy_package_dry_run` reports what it would do, without applying it.

:::warning
A role which is unlocked and not defined by the policy package is deleted, with its permissions and assignments.
Lock the roles managed in Rucio, which is the default for new roles.
:::

### With an identity provider

The roles of an account can be synchronised with the roles an identity provider (IdP) supplies for it,
through `PUT /roles/accounts/<account>` or `RoleClient.sync_account_roles`. Only root and admins can do it.
Rucio does not fetch the roles from the IdP: the caller supplies them.

```json
{"roles": {"analysts": null, "production": "2027-01-01T00:00:00"}, "dry_run": false}
```

`roles` maps each role to the expiry date of its assignment, `null` for none.
It may also be a list of role names and of `{"role": ..., "expires_at": ...}` objects.
Dates are in UTC, in ISO 8601 or in the Rucio date format, e.g. `Fri, 01 Jan 2027 00:00:00 UTC`.
The synchronisation:

1. removes the expired assignments of the account;
1. removes the roles the account holds and the IdP does not supply, including those assigned in Rucio;
1. assigns the supplied roles the account does not hold, active from now on;
1. updates the expiry date of the supplied roles the account holds.

Roles which are not assignable are left untouched. Supplied roles unknown to Rucio are ignored,
as are supplied expiry dates which have already passed, or are not after the activation date of the assignment.
The synchronisation is applied in a single transaction, and returns a report of what was done.
With `dry_run`, it only reports what it would do.

## Upgrading

### Before upgrading Rucio

:::warning
Rucio checks the new read actions of the permission policy whether RBAC is enabled or not.
With a policy package refusing the actions it does not know, reading DIDs, their metadata, content, replicas, rules or locks
is refused to every account but root and admins, even with RBAC disabled.
Before upgrading, define these actions in the policy package, see
[New actions](operator/policy_packages/policy_package_development.md#new-actions).
:::

### Before enabling RBAC

1. If the instance uses a policy package, update its overrides of the DID actions which now apply roles, see
   [Overriding the DID actions](operator/policy_packages/policy_package_development.md#overriding-the-did-actions).
1. Create the roles, grant them their DID permissions, and assign them to the accounts which need them,
   in particular read on the DIDs needed for the [operations requiring to read DIDs](#operations-requiring-to-read-dids).
1. Enable RBAC, see [Enabling RBAC](#enabling-rbac).
