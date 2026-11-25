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

Before enabling RBAC on an existing instance, follow [Upgrading to RBAC](#upgrading-to-rbac).

## Concepts

### Roles

A role has a name, an optional description and two flags:

- `assignable`: whether the role can be assigned to and removed from accounts.
  A role which is not assignable can only be assigned or removed with `force`.
- `locked`: whether the role is protected from changes. Changing the description,
  the `assignable` flag or the permissions of a locked role, or deleting it, requires `force`.
  Unlocking it does not.

A role still assigned to accounts can only be deleted with `force`, which also removes it from these accounts and drops its permissions.

The `admin` role is reserved. It is created with the database schema, not assignable, is always locked,
and can neither be deleted nor be granted permissions: only its `assignable` flag can be changed.
While RBAC is enabled, an account holding the `admin` role is an admin, as is an account with the `admin` account attribute.
The attribute keeps working, so enabling RBAC demotes no admin.

### Role assignments

A role is assigned to an account with an optional activation date (`activates_at`, now by default)
and an optional expiry date (`expires_at`, none by default).
An assignment grants the role only while it is active: from its activation date until its expiry date.
An expired assignment grants nothing, but stays in the database until it is removed.

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

### Operations requiring to read DIDs

With RBAC enabled, the following operations require the account to read the DIDs they involve.
Before, any account could perform them on any DID:

- adding a replication rule of its own on a DID, including the rules given when creating a DID.
  A rule copies the whole content of a collection, so the account must also read all of it, recursively:
  every container, dataset and file the DID contains when the rule is created;
- adding a replica of an existing file on a user RSE;
- attaching DIDs to a collection, even to one in a scope the account owns;
- getting the transfer requests of a DID.

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

## Upgrading to RBAC

1. If the instance uses a policy package, update it as described in
   [Policy packages and RBAC](operator/policy_packages/policy_package_development.md#policy-packages-and-rbac):
   define the new actions, and update the overrides of the DID actions which now apply roles.
1. Create the roles, grant them their DID permissions, and assign them to the accounts which need them,
   in particular read on the DIDs needed for the [operations requiring to read DIDs](#operations-requiring-to-read-dids).
1. Enable RBAC, see [Enabling RBAC](#enabling-rbac).
