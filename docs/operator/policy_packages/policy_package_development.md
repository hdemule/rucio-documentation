---
id: policy-package-development
title: Developing a Policy Package
---

## Template repository
A template policy package repository can be found [here](https://github.com/rucio/policy-package-template).

## Project structure
The basic elements of a policy package are the following:

- An `__init__.py` file that:
  - indicates the supported Rucio version via the `SUPPORTED_VERSION` field;
  - indicates the algorithms provided by the package (optional)
- A `permission.py` module implementing permission
  customisations (optional).
- A `schema.py` module implementing schema customization (optional).
- One or more files for experiment-specific algorithms (optional).

The recommended Python package layouts can be found [here](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/). An example `src`-layout based policy package is as such:

```
experiment-rucio-policy-package
│   README.md
│   pyproject.toml
│
└───src
│   │
│   └───experiment-rucio-policy-package
│       │   __init__.py               # required
│       │   permission.py             # optional
│       │   schema.py                 # optional
│       │   pfn2lfn.py                # optional (deterministic scope translation algorithm)
│       │   non_deterministic_pfn.py  # optional (non-deterministic scope translation algorithm)
│       │   ...
```


### `__init__.py`

#### `SUPPORTED_VERSION`

`__init__.py` should define a `str | list[str]` called `SUPPORTED_VERSION`,
indicating the version(s) of Rucio that your package supports.

This is checked against the Rucio server
version to ensure compatibility when loading the policy package.

##### From Rucio 36
From Rucio 36, version checking was modified
to use [PEP-compliant version specifiers](https://peps.python.org/pep-0440/#version-specifiers).

For example, to specify support for the entire Rucio 36 release line (so 36.1.0, 36.2.0...)
without yet supporting Rucio 37,
the [**compatible release** operator](https://peps.python.org/pep-0440/#compatible-release) `~=`
can be used, as seen below.

```python
SUPPORTED_VERSION = '~=36.0'
```

Multiple constraints can be specified, either as a string:

```python
SUPPORTED_VERSION = '~=36.0,!=36.4.0'
```

Or as a list:

```python
SUPPORTED_VERSION = ['~=36.0','!=36.4.0']
```

##### Before Rucio 36

On Rucio versions older than 36, only major versions can be specified.
This can be done as either a string:

```python
SUPPORTED_VERSION = '35'
```

Or as a list, to indicate support for multiple major versions:

```python
SUPPORTED_VERSION = ['34', '35']
```

#### `get_algorithms`

The `__init__.py` file can also contain
an optional function called `get_algorithms` that
returns a dictionary of custom algorithms implemented within the package.
In fact, this structure should be a "dictionary of dictionaries" where
the outer dictionary contains algorithm types, and each inner
dictionary contains all the algorithms provided by the package for that
type. Currently supported types are `lfn2pfn` for generating PFNs for
deterministic storage, `non_deterministic_pfn` for generating PFNs for
non-deterministic storage, and `scope` for scope extraction algorithms.
(For backwards compatibility, `surl` can be used in place of
`non_deterministic_pfn`, however this is not recommended for new policy
packages).

> [!NOTE]
> Some base algorithm classes depend on `schema` being loaded.
> To avoid **circular import** issues,
> import the algorithm classes as part of the `get_algorithms` function,
> to ensure that `schema` is loaded first.

Example:

```python
def get_algorithms():
    from vo_policy_package.non_deterministic_pfn import VONonDeterministicPFNAlgorithm
    from vo_policy_package.lfn2pfn import VORSEDeterministicTranslation
    from vo_policy_package.scope import VOScopeExtractionAlgorithm
    return {
      'non_deterministic_pfn': {
        'voname_non_deterministic_pfn': VONonDeterministicPFNAlgorithm.construct_non_deterministic_pfn_voname
        },
      'lfn2pfn': {
        'voname_lfn2pfn': VORSEDeterministicTranslation.lfn2pfn_voname
        },
      'scope': {
        'voname_extract_scope': VOScopeExtractionAlgorithm.extract_scope_voname
        }
      }
```

In all cases the names used to register the functions (e.g. `voname_extract_scope`) must be prefixed
with the name of the virtual organisation that owns the policy package,
to avoid naming conflicts on multi-VO Rucio installations.

See [the Algorithms documentation page](operator/policy_packages/policy_package_algorithms.md) for more on developing algorithms.

### Permission and schema modules

The `permission.py` and `schema.py` modules are optional; an experiment
that does not need to customise these modules can omit one or both of
them from the policy package and the Rucio generic versions will be
used instead. If these modules are required, the easiest way to create
them is to modify the generic versions from the Rucio codebase. These
can be found in
[`lib/rucio/core/permission/generic.py`](https://github.com/rucio/rucio/blob/master/lib/rucio/core/permission/generic.py)
and [`lib/rucio/common/schema/generic.py`](https://github.com/rucio/rucio/blob/master/lib/rucio/common/schema/generic.py) respectively.

The `has_permission` function in the permission module may return `None`
if your experiment does not implement a custom permission check for a
particular action. In this case, the generic permission module will be
called for this action instead.

The schema module of a policy package does not need to define all of
the schema values. Any missing ones will automatically be loaded from
the generic schema module instead.

The schema has a hierarchical structure. If you wish to reference a
schema value from within another schema value, you can use the
`SchemaRef` class. This class ensures that schema values are loaded
from the appropriate place, even when some members of the hierarchy
are overridden in the policy package and others are not.

For example, if you wanted to increase the maximum number of items
in an array of accounts to 2000 but reuse the default definition of
`ACCOUNT` from the generic schema, you could refer to it using
`SchemaRef`:

```python
from rucio.common.schema.schema_ref import SchemaRef

ACCOUNTS = {"description": "Array of accounts",
            "type": "array",
            "items": SchemaRef("ACCOUNT"),
            "minItems": 0,
            "maxItems": 2000}
```

You can find the generic schema at [`lib/rucio/common/schema/generic.py`](https://github.com/rucio/rucio/blob/master/lib/rucio/common/schema/generic.py).

## Policy packages and RBAC

[Role-based access control](operator/role_based_access_control.md) (RBAC) is implemented in the generic permission module.
A policy package with its own `permission.py` module must be checked before upgrading Rucio and before enabling RBAC.

### New actions

The following actions are checked by Rucio and must be granted by the permission module:

- reading DIDs: `list_dids`, `list_parent_dids`, `get_did`, `get_metadata`, `get_metadata_bulk`, `list_content`,
  `list_content_history`, `list_files`, `scope_list`, `list_archive_content`, `get_users_following_did`, `add_did_to_followed`;
- reading rules and locks: `list_replication_rule_history`, `list_replication_rule_full_history`, `get_replication_rule`,
  `examine_replication_rule`, `get_dataset_locks`, `get_dataset_locks_bulk`, `list_associated_replication_rules_for_file`,
  `get_replica_locks_for_rule_id`;
- reading replicas: `list_replicas`, `list_dataset_replicas`, `list_dataset_replicas_bulk`, `list_dataset_replicas_vp`;
- deleting a metadata key: `delete_metadata`.

The generic module maps the read actions to `perm_read_data` and `delete_metadata` to `perm_delete_metadata`.
Both grant every account while RBAC is disabled.

If the `has_permission` function of the policy package returns `None` for the actions it does not know,
Rucio falls back to the generic module and nothing needs to be done.
If it resolves them to its own default instead, e.g. a `perm_default` granting only root and admins,
these operations are refused to every other account. Map them explicitly:

```python
from rucio.core.permission.generic import perm_delete_metadata, perm_read_data

perm = {
    # ... the actions of the policy package
    'list_dids': perm_read_data,
    'get_metadata': perm_read_data,
    'list_replicas': perm_read_data,
    # ... and the other read actions listed above
    'delete_metadata': perm_delete_metadata,
}
```

### Overriding the DID actions

With RBAC enabled, the following generic actions apply the DID permissions of roles, and require reading the DIDs they involve:
`add_did`, `add_replicas`, `add_rule`, `attach_dids`, `attach_dids_to_dids`, `create_did_sample`, `detach_dids`,
`get_request_by_did`, `set_metadata` and `set_metadata_bulk`.

A policy package overriding one of them replaces the generic function: roles then grant nothing on that action,
and reading the DIDs it involves is not required. Since RBAC hides the DIDs an account may not read,
such an override would let the account act on them anyway, e.g. add a rule on a hidden DID, or on a collection
whose content it may not all read, and so copy it to an RSE.
Before enabling RBAC, make these overrides call the generic function, or mirror it.

### Listing filters

With RBAC enabled, listings are filtered item by item: each item is kept only if the account may read its DID or its scope.
A permission module may define the following functions to change how listings are filtered.
Each function it does not define falls back to the generic one:

- `is_listing_filtered(issuer, action, session)`: whether the listing named `action` is filtered for the account.
  The generic function filters only with RBAC enabled.
- `scope_filter(issuer, action, session, operation=DidOperationType.READ, any_did=False)`:
  returns a callable taking a scope, telling whether the account may perform `operation` on the whole scope,
  or with `any_did`, on some DIDs of the scope.
- `did_filter(issuer, action, session, operation=DidOperationType.READ)`:
  returns a callable taking a scope and a name, telling whether the account may perform `operation` on the DID.

The callables are built once per listing, so that a long listing does not query the database for each item.
