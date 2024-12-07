+++
title = "Two years in the Vault: 4 best practices"
date = 2024-12-06T19:35:04+01:00
draft = false
toc = true
tags = ["hashicorp vault", "security", "platform engineering", "devops", "secrets management"]
description = "Sharing learnings about the use of [HashiCorp Vault](https://www.vaultproject.io/) for managing secrets in a platform engineering context."
+++

I work as an IT consultant. Over the past two years, we've been working with our
client on a
[platform engineering](https://en.wikipedia.org/wiki/Platform_engineering)
project, where the use of [HashiCorp Vault](https://www.vaultproject.io/) for
[secrets management](https://www.redhat.com/en/topics/devops/what-is-secrets-management)
was prevalent. Our Vault has enabled us to manage hundreds of credentials,
increasing the security of our developer platform, and has resulted in a
thorough and battle-tested configuration of our Vault which we can be proud of.

However, over the past two years, there have been quite a few learning moments.
Vault is a great product with lots of flexibility, but this opens up a lot of
possibilities for misconfiguration or misuse. For us, that meant a few tedious
reconfigurations of our Vault. This post aims to share learnings that I wish we
had known from the very beginning, in the form into digestible tips.

These tips are for everyone that needs to work with HashiCorp Vault, whether it
be as a developer, or as an administrator (you may still learn something). If
you don't use Vault, maybe simply bookmark this post, it may come in handy in
the future.

TL;DR:

1. Configure with an infrastructure-as-code (IaC) tool
1. Policies describe permissions
1. KVv2: save raw data, format elsewhere
1. Beware of write permissions on `sys/policy`
1. Consider templated policies (upcoming)
1. KVv2: make secrets "connection bundles" (upcoming)
1. Utilize `-output-policy` from the CLI (upcoming)
1. Think about your path structure (upcoming)

This is a two-part series, we'll cover the first four tips here, and the
remaining four in an upcoming post. Stay tuned.

> **Glossary:** I overuse the words _component_ and _system_ a lot. This could
> refer to microservices, short running jobs, monolith servers, which all run
> together, to form some sort of workload. If you're running on Kubernetes,
> think of a _component_ as a Deployment, and a _system_ as the Kubernetes
> cluster.

## Tip 1: Configure with an infrastructure-as-code (IaC) tool

When first starting to integrate a Vault into a large system, one naturally does
explorative work locally with the Vault CLI. This is all good, but concretizing
that configuration into infrastructure-as-code (IaC) is essential, and by doing
it early you'd avoid the pain of migrating your configuration down the road.

Potential solutions for IaC are:

- [Terraform](https://www.terraform.io/)/[OpenTofu](https://opentofu.org/)
- [Pulumi](https://www.pulumi.com/)

For configuring your Vault, I would generally recommend going for
Terraform/OpenTofu, which takes the declarative approach as opposed to Pulumi.
Pulumi is a good solution when you have lots of dependencies in a complex
system, which need to be handled programmatically.

We started off on our Vault journey by simply documenting how we configured our
secrets engines, auth methods and policies. So when we had to set up a new
Vault, that meant going through that documentation and running the commands
one-by-one. We at some point (painfully) changed to Terraform/OpenTofu, and it
directly opened up many doors for us:

- Writing a testing framework, where we would test the access to Vault paths by
  simulating the components in our system, allowing us to check for any
  regressions
- Reusing the IaC to configure Vaults in multiple environments
- Integrate with CI pipelines to automate updates to our Vaults

There's many more advantages. Point is, start with IaC directly, so you don't
have to go through the same process as we did, of integrating an already
configured Vault with IaC. In case you have to:
[Terraform Import Blocks](https://developer.hashicorp.com/terraform/language/import)
saved our lives :)

## Tip 2: Policies describe permissions

Policies are core to HashiCorp Vault, being one of the pillars in its access
control mechanisms. A Vault policy is a set of permissions, each being a
combination of a path and capabilities on that path. Efficient management of
policies is important in order to keep your Vault lean and scalable.

Let's consider a very basic Vault, which has a
[KVv2 engine](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2) under
`secrets/`, used to store secret recipes under the `recipes/` subpath. The
following policy could be the "head chef" policy, allowing the head chef to
browse the entire secrets engine and update any of the secret recipes:

```bash
vault policy write head-chef - <<EOF
# browse secrets
path "secrets/metadata/*" {
  capabilities = ["read", "list"]
}
# manage recipes
path "secrets/data/recipes/*" {
  capabilities = ["create", "update", "read", "delete"]
}
EOF
```

We would then attach this policy to a role with:

```
vault write auth/approle/role/head-chef token_policies=head-chef
```

Here we decided that the name of the policy would match the role, namely
`head-chef`. This works, but we have found that as policies become larger and
more roles are added to the Vault, **breaking down policies into smaller, more
meaningful sets of permissions** becomes increasingly important. **Policy names
should also describe the permissions encapsulated**, and should not refer back
to the role that uses it.

To help breaking down policies, it is useful to decide on a **naming standard**
for your policies. An example would be `<verb>-<subject of policy>`, like
`read-nuclear-codes`. In this example we would have to break down the policy a
bit more in order for our convention to apply. Let's split the `head-chef`
policy into two policies, `browse-secrets` and `manage-recipes`:

```bash
vault policy write browse-secrets - <<EOF
# browse secrets
path "secrets/metadata/*" {
  capabilities = ["read", "list"]
}
EOF
vault policy write manage-recipes - <<EOF
# manage recipes
path "secrets/data/recipes/*" {
  capabilities = ["create", "update", "read", "delete"]
}
EOF
vault write auth/approle/role/head-chef token_policies=browse-secrets,manage-recipes
```

It is now much more transparent what permissions are associated with this role.
In addition, breaking down the policy made parts of the policy reusable, as we
could now do something like the following, reusing the `browse-secrets` policy:

```bash
vault policy write manage-greek-recipes - <<EOF
# manage greek recipes
path "secrets/data/recipes/greek/*" {
  capabilities = ["create", "update", "read", "delete"]
}
EOF
vault write auth/approle/role/chef-george token_policies=browse-secrets,manage-greek-recipes
```

Deciding on this naming convention for our policies allowed us to scale to many
more roles in our Vault. One thing to consider though, is that paths may end up
overlapping among the policies assigned to a role. This requires being aware of
how Vault's
[priority matching](https://developer.hashicorp.com/vault/docs/concepts/policies#priority-matching)
works.

## Tip 3: KVv2: save raw data, format elsewhere

Vault's KVv2 engine allows for saving static secrets in the form of key-value
pairs. Choosing the appropriate key-value pairs to save in a secret will both
maximize flexibility in how your infrastructure consumes these secrets and would
minimize duplicated data.

Let's say you have a component which requires credentials to a database, and
needs them provided inside a config, e.g. in TOML format. Since the config holds
secrets, it may be tempting to save the entire config in Vault's KVv2 engine,
and fetch that config when starting your component:

```bash
vault kv put secrets/components/my-component config.toml=- <<EOF
[database]
host = "database.example.com"
username = "postgres"
password = "1234"
EOF
```

However, this approach is limiting:

- Reusing the database credentials means copy-pasting to another config, and
  doesn't allow for a source of truth
- Any non-secret config changes need to go over the Vault

This can be avoided: the Vault ecosystem has a large amount of helper components
which support templating, among which:

- [Vault agent templating](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template)
- [External Secrets Operator templating](https://external-secrets.io/latest/guides/templating/)
  in case you're running on Kubernetes.

As an example let's make use of an `ExternalSecret` for our templating. First,
let's save the **standalone credentials** instead of our config file:

```bash
vault kv put secrets/databases/my-database username=postgres password=1234
```

Now we'll write an `ExternalSecret` making use of this raw data:

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: config
spec:
  # ... (reference to SecretStore)
  target:
    template:
      engineVersion: v2
      data:
        config.toml: |
          [database]
          host = "database.example.com"
          username = {{ .username | quote }}
          password = {{ .password | quote }}
  dataFrom:
  - extract:
      key: /secrets/databases/my-database
```

This would then render a `Secret` resource with the following content under the
`config.toml` key, which can be mounted in our component:

```toml
[database]
host = "database.example.com"
username = "postgres"
password = "1234"
```

The structure of our KVv2 engine is now cleaner, as it is not aware of the
components that access it, and holds sensible reusable secrets. Another option
is to tweak how the component is configured. Instead of taking credentials
directly, our component could take a Vault path instead:

```toml
[database]
host = "database.example.com"
vault_path = "secrets/databases/my-database"
```

This however means a dependency on HashiCorp Vault in the code of the
application itself, as the code now uses the Vault API in order to retrieve the
secret. Some applications may want to be agnostic to the type of software
storing the credentials, in which case a templating solution would be more
suitable.

## Tip 4: Beware of write permissions on `sys/policy`

> Understanding this tip is a bit more involved, but please bear with me. It is
> arguably the most important tip in this series, especially in environments
> where security is critical.

Vaults used in a complex system sometimes need to support new users being
asynchronously added to the Vault: for example adding a new role/user which has
access to a subdirectory of a KVv2 secrets engine. To automate this, companies
may resort to building some management component which automates the creation of
a role/user and policies attached to it.

The
[`sys/policy`](https://developer.hashicorp.com/vault/api-docs/system/policy#create-update-policy)
(or
[`sys/policies/acl`](https://developer.hashicorp.com/vault/api-docs/system/policies#create-update-acl-policy))
path is used to grant permissions on managing policies in Vault, and is
necessary to create policies on the fly. We will see in this chapter how
granting write permissions on `sys/policy` **can lead to severe security
risks**, and provide some examples on how to mitigate these risks.

As an example, let's consider our Vault which is meant to manage recipes. Our
company wants to support employees requesting their own subdirectory in our
`secrets/` KVv2 engine. This would involve the employees sending an API request
to a component, let's call it the `employee-manager`, that would need to execute
following commands on the Vault upon receiving one of these requests:

```bash
# create policy for employee
vault policy write "manage-recipes-$EMPLOYEE_NAME" - <<EOF
path "secrets/data/recipes/$EMPLOYEE_NAME/*" {
  capabilities = ["create", "read", "update", "delete"]
}
EOF
# create role for employee with attached policy
vault write auth/approle/role/$EMPLOYEE_NAME token_policies="manage-recipes-$EMPLOYEE_NAME"
```

This would require the `employee-manager` component to have the necessary
permissions to create both roles and policies for our employees. Let's setup a
role and policy to allow the component to execute the above commands:

```bash
# create "create-policies", "create-roles" policies
vault policy write "create-policies" - <<EOF
path "sys/policy/+" {
  capabilities = ["create", "update"]
}
EOF
vault policy write "create-roles" - <<EOF
path "auth/approle/role/+" {
  capabilities = ["create", "update"]
}
EOF
# attach policies to role
vault write auth/approle/role/employee-manager token_policies=create-policies,create-roles
```

The above permissions are **extremely dangerous** to provide to any component or
user of Vault, especially in conjunction.

Imagine our `employee-manager` component was insecure, and is compromised by an
attacker. The attacker then uses the component's credentials to log in to Vault.
They can now **create new policies, with any permissions, and assign those
permissions to the `employee-manager` which they have control over**:

```bash
# create malicious policy
vault policy write "read-secrets" - <<EOF
path "secrets/data/*" {
  capabilities = ["read"]
}
EOF
# update the compromised role with malicious policy
vault write auth/approle/role/employee-manager token_policies=create-policies,create-roles,read-secrets

# after logging in again, the attacker could execute:
vault read secrets/data/recipes/gordon/super-secret-meatballs-recipe
```

So the component must technically be considered as **admin** on the Vault, as it
can create policies for absolutely anything and assign it to itself. To mitigate
this security risk, there are the following solutions:

1. Avoid granting permissions on `sys/policy` altogether and consider **using
   [templated policies](https://developer.hashicorp.com/vault/docs/concepts/policies#templated-policies)**.
   This is often the cleanest solution, and has a big advantage of reducing the
   amount of configuration.
1. Grant permissions on a **subfolder of `sys/policy`**. This subfolder must be
   disjoint from the policies granted to `employee-manager`, and
   `employee-manager` should not be able to `update` its own role.
1. Grant only `create` permissions on anything under `sys/policy`, and
   `employee-manager` should not be able to `update` its own role.

There may be more mitigation options, but the point of this chapter is to
consider the event of a breach and be aware of the **effective permissions**
that you may be giving to your system components.

That's all the tips for now. Thanks for reading, I hope you learned something,
and stay tuned for the second half and other posts to come.
