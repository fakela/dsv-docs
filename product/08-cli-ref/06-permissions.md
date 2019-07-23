﻿[title]: # (Permissions)
[tags]: # (,)
[priority]: # (1860)

# Permissions

DevOps Secrets Vault configs are all about permissions. To see the current permissions policies, you read out the contents of the config:

```bash
thy config read -be yaml
```

You should see a result similar to this:

```yaml
permissionDocument:
- id: bgn8gjei66jc7148d9i0
  actions:
  - <.*>
  conditions:
    remoteIP:
      type: CIDRCondition
      options:
        cidr: 192.168.0.1/16
  description: Default Admin Policy
  effect: allow
  resources:
  - <.*>
  subjects:
  - users:<aws-dev:test-admin|admin>
  - roles:<administrator>
- id: erji23829823
  description: Developer Policy.
  subjects:
  - users:<developer1@thycotic.com|developer2@thycotic.com>
  actions:
  - "<read|delete|create|update|share>"
  effect: allow
  resources:
  - secrets:servers:us-east-1:<.*>
- id: erji23829825
  description: Developer Deny Policy.
  subjects:
  - users:<developer1@thycotic.com|developer2@thycotic.com>
  actions:
  - "<.*>"
  effect: deny
  resources:
  - secrets:servers:us-east-1:production:<.*>
- id: erji23829828
  description: Role Assignment Policy.
  subjects:
  - users:role.admin@thycotic.com
  - roles:<admin-role>
  actions:
  - "<assign>"
  effect: allow
  resources:
  - roles:<.*>
- id: erji23829829
  description: Client Credentials Create Policy.
  subjects:
  - users:<client.admin@thycotic.com|developer1@thycotic.com>
  actions:
  - "<read|delete|create|update>"
  effect: allow
  resources:
  - clients:<.*>
settings: null
tenantName: example
```

The `permissionDocument` lists policies that define a subject’s access to a certain resource path. The syntax supports wildcards via the `<.*>` construct.
  
---
  
| Element | Definition |
|----- | ----- |
| actions | a list of possible actions on the resource (create, read, update, delete, list, share, assign) |
| conditions | an optional CIDR range to lock down access to a specific IP range |
| description | human friendly description of the policy intent |
| effect | whether the policy is allowing or preventing access; valid values are `allow` and `deny` |
| id | system-generated unique identifier to track changes to a particular policy |
| resources | the resource path defining the entities to which the permissions apply; a resource path prefixes the entity type (`secrets, clients, roles, users`) to a colon delimited path to the resource. |
  
---
  
## Policy Evaluation

To correctly evaluate permission policies, you must know the rules that apply to permissions.

* Permissions are cumulative.

  If there is a top level permission for the path `secrets:servers:<.*>` that grants a user write access, then even if they are only granted read access at the resource path `secrets:servers:webservers:<.*>`, they will still have write access due to the top level implicit match. 

* An explicit `deny` trumps an explicit or implicit `allow`.

* Actions are explicit. A user assigned `update` and `read` will not automatically have `create` for the resource path.

* The `list` action has a special behavior.

  * First, `list` (search) is global—it runs across all items of an entity, not limited to paths and sub-paths.

  * Second, to grant a user an ability to search entities via `list`, use the root of the entity if you want `list` to include other entities and actions within the same policy. The root entity, for example, is `secrets`, with no other characters following.

## Policy Examples

### Deny Access at a Lower Level

**Case:** Subjects need access to secrets for an environment, but that logical environment contains a more restricted area.

**Solution:** Add an explicit `deny` with a more specific path to allow general access to the secrets, but deny access at the more privileged level, as in the following example.

```yaml
- id: erji23829823
  description: Developer Policy.
  subjects:
  - users:<developer1@thycotic.com|developer2@thycotic.com>
  actions:
  - "<read|delete|create|update|share>"
  effect: allow
  resources:
  - secrets:servers:us-east-1:<.*>
- id: erji23829825
  description: Developer Deny Policy.
  subjects:
  - users:<developer1@thycotic.com>
  actions:
  - "<.*>"
  effect: deny
  resources:
  - secrets:servers:us-east-1:production:<.*>
```

### Allow Users to Assign Specific Roles

**Case:** A user needs to assign roles when they create client credentials, but must not be able to self elevate by assigning an admin level role.

**Solution:** Use a naming convention when creating roles and specify a prefix with a wildcard to only allow users to assign roles that match the naming convention, as modeled in the following example.

```yaml
- id: erji23829828
  description: Limited Role Assignment Policy.
  subjects:
  - users:developer@thycotic.com
  - roles:onboarding-role
  actions:
  - "<assign>"
  effect: allow
  resources:
  - roles:dev-role-<.*>
```

### Allow Users to List Specific Entities

**Case:** A user needs to `read` and `list` entities within a policy.

**Solution:** Add a `list` action and the root of the entity used for searching. 

In the example below, `roles` is the entity for reading and searching (`list` action). In the resources section,
`roles:dev-role-<.*>` is used for reading, while `roles` is used for searching.

```yaml
- id: erji23829828
  description: Limited Role Policy.
  subjects:
  - users:developer@thycotic.com
  - roles:onboarding-role
  actions:
  - "<read|list>"
  effect: allow
  resources:
  - roles:dev-role-<.*>
  - roles
```

The syntax of the latter is important. In general, the root form of an entity has no `*` after the entity name, or anything besides the name.
