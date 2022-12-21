---
title: What is tf.libsonnet
description: Learn about tf.libsonnet and advantages of using Jsonnet for generating Terraform
no_list: true
weight: 1
---

This document walks through an overview of Jsonnet and `tf.libsonnet`, as well as some of the reasons you might consider using
it for your next Terraform project.

## What is Jsonnet?

[Jsonnet](https://jsonnet.org) is a data templating language originally created by Google. It is a superset of JSON that
adds programming constructs to the language. Think of it as JSON enhanced with:

- [Variables](https://jsonnet.org/learning/tutorial.html#variables)
- [Conditionals](https://jsonnet.org/learning/tutorial.html#conditionals)
- [Functions](https://jsonnet.org/learning/tutorial.html#functions)
- [Iteration](https://jsonnet.org/learning/tutorial.html#comprehension)

Jsonnet is designed and optimized for the use case of writing configuration files. Check out [their design
rationale document](https://jsonnet.org/articles/design.html) for more details.

## What is tf.libsonnet?

`tf.libsonnet` is a collection of pure Jsonnet libraries that provide functions and utilities for generating
[Terraform](https://www.terraform.io/) code (as [JSON configuration
syntax](https://developer.hashicorp.com/terraform/language/syntax/json)). The libraries allow you to streamline the
experience of writing static Terraform JSON configuration.

Here is an example:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';
local tfnull = import 'github.com/tf-libsonnet/hashicorp-null/main.libsonnet';

local o =
  tf.withProvider('null', {}, src='hashicorp/null', version='~>3.0')
  + tf.withVariable('trigger_id')
  + tfnull.resource.new(
    'this',
    triggers={
      trigger_id: '${var.trigger_id}',
    },
  )
  + tf.withOutput(
    'this_id',
    o._ref.null_resource.this.get('id'),
  );

o
```

This is equivalent to the following Terraform JSON code:

```json
{
   "terraform": {
      "required_providers": {
         "null": {
            "source": "hashicorp/null",
            "version": "~>3.0"
         }
      }
   },
   "provider": {
      "null": [
         { }
      ]
   },
   "variable": {
      "trigger_id": { }
   },
   "resource": {
      "null_resource": {
         "this": {
            "triggers": {
               "trigger_id": "${var.trigger_id}"
            }
         }
      }
   },
   "output": {
      "this_id": {
         "value": "${null_resource.this.id}"
      }
   }
}
```


## Comparison to Terraform HCL

### Limitations of HCL

As a configuration language, Jsonnet is very similar to HCL. Both languages strive to enhance the experience of writing
static declarative configuration files in `YAML` or `JSON`.

That same example above can be written in Terraform HCL in a more terse and readable format:

```hcl
terraform {
  required_providers {
    null = {
      source  = "hashicorp/source"
      version = "~> 3.0"
    }
  }
}

variable "trigger_id" {}

resource "null_resource" "this" {
  triggers = {
    id = var.trigger_id
  }
}

output "this_id" {
  value = null_resource.this.id
}
```

HCL provides many of the same constructs as Jsonnet, which means that there isn't much to gain from using Jsonnet for a
single Terraform module.

However, using Jsonnet can greatly increase the maintainability of large scale Terraform deployments that span multiple
modules and state files. When expanding a Terraform project to many smaller modules, you run into many limitations of
the Terraform HCL language that is caused by its tight coupling to the Terraform runtime.

To name a few:

- Code reusability is limited to a single module, and thus single state file.
- Certain constructs can not be interpolated dynamically (e.g.,
  [lifecycle](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle#literal-values-only) and
  [backend](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)).
- Certain blocks can not be reused across modules (e.g.
  [provider](https://developer.hashicorp.com/terraform/language/modules/develop/providers)).
- A single module can not write to multiple state files.

### Motivating example

Consider a Terraform deployment where you have a single database in a VPC, across two environments (`stage` and `prod`).
To simplify the example, we will also assume we have defined sub modules for defining a canonical VPC and MySQL
database. This will reduce the root modules in our examples to a single module block.

In this example, we will assume that we want to follow best practices and [isolate the state
files](https://blog.gruntwork.io/how-to-manage-terraform-state-28f5697e68fa#784f) for the components. This will result
in four state files:

- VPC in Stage
- VPC in Prod
- MySQL database in Stage
- MySQL database in Prod

To achieve this, we need to define the components with four root modules, one for each state file above. We will have a
project structure like below:

```text
.
├── stage
│   ├── mysql
│   │   ├── backend.tf
│   │   ├── main.tf
│   │   └── provider.tf
│   └── vpc
│       ├── backend.tf
│       ├── main.tf
│       └── provider.tf
└── prod
    ├── mysql
    │   ├── backend.tf
    │   ├── main.tf
    │   └── provider.tf
    └── vpc
        ├── backend.tf
        ├── main.tf
        └── provider.tf
```

The `main.tf` file for each component contains a single module call to define the underlying component infrastructure
with some hardcoded parameters as the inputs.

For example, the `stage/vpc/main.tf` might look like:

```hcl
module "vpc" {
  source = "github.com/myorg/my-vpc?ref=v1.0.8"

  name       = "stage"
  cidr_block = "10.0.0.0/16"
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

The `provider.tf` file will contain the provider configuration with the `required_providers` block to specify
the version:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}
```

Finally, the `backend.tf` file will contain the backend configuration for storing the state file in S3. For example, for
`stage/vpc`:

```hcl
terraform {
  backend "s3" {
    bucket = "my-stage-bucket"
    key    = "vpc/terraform.tfstate"
    region = "us-west-2"
  }
}
```

This setup works well for a small scale deployment like above, or for a simple provider and backend configuration.

But what if you have hundreds of components distributed across tens of environments? And what if you need to adjust your
provider configurations to use different AWS IAM Roles or you want to enhance it with `allowed_account_ids`?

The challenge is that you can’t modularize the `provider` and `backend` configurations due to the aforementioned
limitations. This results in the `provider.tf` and `backend.tf` files being mostly duplicated across all the modules,
and maintained manually, leading to a painful refactoring process if you ever need to change a pattern in one of these
files.

We can address this by using Jsonnet.

### Using Jsonnet to DRY multi-component, multi-environment Terraform projects

In the previous example we discussed the limitations of Terraform HCL around DRY-ing certain components. Now let's see
what we can accomplish with Jsonnet.

> **INFO**:
>
> All the code is available at https://github.com/tf-libsonnet/infrastructure-live-example.

First, since Jsonnet is not bound by Terraform, we can essentially modularize any component using functions. Even though
Terraform may not support this, in Jsonnet we can create a function that generates the `provider` and `backend` blocks
and reuse it wherever we need it:

**_provider.libsonnet_**

```text
local const = import './constants.json';
local aws = import 'github.com/tf-libsonnet/hashicorp-aws/main.libsonnet';

aws.provider.new(region=const.region)
```

**_backend.libsonnet_**

```text
local const = import './constants.json';

{
  getBackend:: function(envName, component) {
    // NOTE: the + syntax in the keys. This indicates a merge operation, which
    // means deep merge the nested object as opposed to replacing the
    // contents.
    terraform+: {
      backend+: {
        s3: {
          bucket: 'my-' + envName + '-bucket',
          key: component + '/terraform.tfstate',
          region: const.region,
        },
      },
    },
  },
}
```

> **NOTE**:
>
> We use the `.libsonnet` extension here. Conventionally, Jsonnet libraries intended to be imported use the `.libsonnet`
> extension while root files that are meant to be processed by `jsonnet` use the `.jsonnet` extension.

In addition to modularizing these blocks, we can also interpolate values in the backend block to dynamically generate
the content based on given parameters. This is possible because the values are static by the time Terraform sees it post
compilation. In this way, we can construct the backend block for each region and component without all the boilerplate
repeated.

We also extracted the hard coded region into a reusable JSON file to promote more DRY code. Since Jsonnet is a superset
of JSON, we can import plain JSON like normal Jsonnet. The contents of `constants.json` looks like follows:

```json
{"region": "us-west-2"}
```

Moving on, let's modularize the VPC and MySQL components by defining a function for generating the relevant blocks:

**_vpc.libsonnet_**

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';

{
  // Given the environment name and cidr block, return the VPC module call with
  // output blocks.
  getVPC:: function(envName, cidrBlock) (
    // Bind the resulting object to a reference so we can refer to self.
    local o =
      tf.withModule(
        'vpc',
        'github.com/myorg/my-vpc?ref=v1.0.8',
        {
          name: envName,
          cidr_block: cidrBlock,
        },
      )
      + tf.withOutput('vpc_id', o._ref.module.vpc.get('vpc_id'));

    o
  ),
}
```

**_mysql.libsonnet_**

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';
local aws = import 'github.com/tf-libsonnet/hashicorp-aws/main.libsonnet';

{
  // Given the environment name, return the VPC data source lookup with mysql
  // module call blocks.
  getMySQL:: function(envName) (
    // Bind the resulting object to a reference so we can refer to self.
    local o =
      aws.data.vpc.new(
        'vpc',
        tags={
          Name: envName,
        },
      )
      + tf.withModule(
        'mysql',
        'github.com/myorg/my-mysql?ref=v1.0.8',
        {
          name: envName,
          vpc_id: o._ref.data.aws_vpc.vpc.get('id'),
        },
      )
      + tf.withOutput('fqdn', o._ref.module.mysql.get('fqdn'));

    o
  ),
}
```

At this point, your folder structure should look like the following:

```text
.
├── jsonnetfile.json
├── jsonnetfile.lock.json
└── stack
    ├── backend.libsonnet
    ├── constants.json
    ├── mysql.libsonnet
    ├── provider.libsonnet
    └── vpc.libsonnet
```

Where the `jsonnetfile.json` and `jsonnetfile.lock.json` files are
[jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler) configurations for installing `tf.libsonnet`. You
can generate this using the following commands:

```text
jb init
jb install github.com/tf-libsonnet/hashicorp-aws@main
# NOTE: we don't need to explicitly install tf-libsonnet/core because
#       it will be pulled as a transient dependency of the hashicorp-aws
#       library.
```

With these building blocks, we can define what a single environment should look like:

**_stack/main.jsonnet_**

```text
local backend = import './backend.libsonnet';
local mysql = import './mysql.libsonnet';
local provider = import './provider.libsonnet';
local vpc = import './vpc.libsonnet';

{
  // getStack returns the Terraform code to deploy the full application stack
  // for a single environment. To facilitate this, this returns the resulting
  // main.tf.json for each folder in the environment stack, which should be
  // extracted by the -m flag of jsonnet.
  getStack:: function(envName, cidrBlock) {
    [envName + '/vpc/main.tf.json']: (
      backend.getBackend(envName, 'vpc')
      + provider
      + vpc.getVPC(envName, cidrBlock)
    ),
    [envName + '/mysql/main.tf.json']: (
      backend.getBackend(envName, 'mysql')
      + provider
      + mysql.getMySQL(envName)
    ),
  },
}
```

Note how the `getStack` function returns an object with keys that look like folder/file structures. This uses a very
useful feature of `jsonnet` that allows [outputting multiple documents from a single jsonnet
program](https://jsonnet.org/learning/getting_started.html#multi). Using this, we can generate our entire
environment/component combos with two function calls:

**_main.jsonnet_**

```text
local stack = import './stack/main.libsonnet';

stack.getStack('stage', '10.0.0.0/16')
+ stack.getStack('prod', '10.1.0.0/16')
```

The final folder structure will look something like this:

```text
.
├── jsonnetfile.json
├── jsonnetfile.lock.json
├── main.jsonnet
└── stack
    ├── backend.libsonnet
    ├── constants.json
    ├── main.libsonnet
    ├── mysql.libsonnet
    ├── provider.libsonnet
    └── vpc.libsonnet
```

You can run this through `jsonnet` to generate the Terraform folder structure:

```text
$ jb install
$ jsonnet -J ./vendor -c -m out main.jsonnet
$ tree ./out
./out
├── prod
│   ├── mysql
│   │   └── main.tf.json
│   └── vpc
│       └── main.tf.json
└── stage
    ├── mysql
    │   └── main.tf.json
    └── vpc
        └── main.tf.json
```

As a sample, here is what `stage/vpc/main.tf.json` looks like:

```json
{
   "module": {
      "vpc": {
         "cidr_block": "10.0.0.0/16",
         "name": "stage",
         "source": "github.com/myorg/my-vpc?ref=v1.0.8"
      }
   },
   "output": {
      "vpc_id": {
         "value": "${module.vpc.vpc_id}"
      }
   },
   "provider": {
      "aws": [
         {
            "region": "us-west-2"
         }
      ]
   },
   "terraform": {
      "backend": {
         "s3": {
            "bucket": "my-stage-bucket",
            "key": "vpc/terraform.tfstate",
            "region": "us-west-2"
         }
      }
   }
}
```

What is neat about this approach is that adding a new environment only requires modifying `main.jsonnet`. E.g., adding a
new dev env:

```text
local stack = import './stack/main.libsonnet';

stack.getStack('dev', '10.2.0.0/16')
+ stack.getStack('stage', '10.0.0.0/16')
+ stack.getStack('prod', '10.1.0.0/16')
```

Additionally, since the resulting code is plain Terraform, the runtime can be handled by anything that supports
Terraform natively. This means that you should be able to use your existing Terraform pipelines without modificatins on
the resulting code to deploy the resulting changes (e.g., [Terraform
Cloud](https://developer.hashicorp.com/terraform/cloud-docs) / [Terraform
Enterprise](https://developer.hashicorp.com/terraform/enterprise), [Atlantis](https://www.runatlantis.io/),
[Spacelift](https://spacelift.io/), [env0](https://www.env0.com/), etc). This can greatly help with incrementally
transitioning to using Jsonnet.

### Summary

To summarize the comparison:

- Terraform HCL has limitations on what you can modularize and reuse.
- Terraform HCL is limited to managing a single state file.
- Jsonnet is not bound by the Terraform runtime, giving you (mostly) free reign on what you can reuse and generate. This
  includes `provider`, `lifecycle`, and `backend` blocks.
- Jsonnet can be made to generate multiple folders, and thus automatically generate multi-state Terraform project
  structures.


## Comparison to other tools

If you have run across these limitations of Terraform HCL, you might have been introduced to other tools in this space
that attempt to solve these problems. To name a few:

- [Terragrunt](https://terragrunt.gruntwork.io/)
- [Pulumi](https://www.pulumi.com/)
- [CDKTF](https://developer.hashicorp.com/terraform/cdktf)
- [Terramate](https://github.com/mineiros-io/terramate)

Each of these tools attempt to solve the problem using similar approaches of using a templating abstraction
(Terragrunt` and Terramate` use an HCL abstraction, while CDKTF and Pulumi uses general purpose programming languages).

However, in addition to being a templating abstraction, these tools also attempt to manage the lifecycle of the
resources, and thus the Terraform runtime. For example, instead of running `terraform plan` and `terraform apply`, you
might run:

- `terragrunt plan` and `terragrunt apply`
- `pulumi up`
- `cdktf deploy`
- `terramate run`

This control gives each of these tools the extensibility to implement feature enhancements that are not provided by
`terraform` natively (e.g., [terragrunt
run-all](https://terragrunt.gruntwork.io/docs/features/execute-terraform-commands-on-multiple-modules-at-once/) and
[dependency
blocks](https://terragrunt.gruntwork.io/docs/features/execute-terraform-commands-on-multiple-modules-at-once/#passing-outputs-between-modules)).

However, there are two significant disadvantages that you trade for this power:

- **Leaky abstractions**. The tools try to abstract away Terraform, but because of the [law of leaky
  abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/), it is impossible to
  completely hide it. Bugs and issues in Terraform frequently bubble up to the abstraction layer, which can
  lead to major confusions around debuggability. To most users, it is not clear when an issue is at the Terraform layer
  or the abstraction layer. In some cases, dropping to Terraform is necessary to debug the issues. Albeit, most of these
  tools provide a way to debug such issues, but it may not feel natural (see [terragrunt
  debugging](https://terragrunt.gruntwork.io/docs/features/debugging/) for example).

- **Poor integration support**. Runtime services need to natively support the runtime and outputs of these tools because
  these tools depend on being invoked directly. This adds a barrier to entry as many runtimes need to make the decision
  on whether they should expend time on developing a native integration. Typically, many services start with native
  Terraform support, and later on add these tools as an afterthought, or may not even support it. For example, Terraform
  Cloud / Terraform Enterprise does not natively support calling anything other than `terraform`. This may prevent your
  team from adopting these tools.

Using Jsonnet addresses these issues by adopting a different paradigm: the Terraform Compiler Pattern. In this approach,
the tool is purely a templating abstraction, and does not control any part of actually running `terraform`. This results
in a clean separation between the Jsonnet world and the `terraform` world.

Here is how that addresses the above disadvantages:

- **Clear separation of boundaries**. Note that using Jsonnet does not preclude you from learning Terraform. You must
  still be aware of Terraform, and the associated runtime. However, by explicitly separating out the steps, we embrace
  the leaky abstraction instead of hiding it. This gives you clear entrypoints for addressing an issue. Need to drop to
  `terraform`? Interact directly with the "compiled" code. Found an issue in the generated `terraform` output? Focus on
  the Jsonnet code to compare the generated outputs.
- **Any runtime/tool that natively supports Terraform is supported**. Because the output is pure Terraform JSON code,
  any tool that works with Terraform can work with the Jsonnet output. The Jsonnet is only a preprocessing step for
  generating the Terraform code, and does not care about how the resulting code is run or deployed. For example, you can
  integrate with Terraform Cloud / Terraform Enterprise by having a secondary repository from your Jsonnet code that
  stores the compiled Terraform code.

Note that some of the aforementioned tools support a similar pattern by focusing on just the templating feature. For
example, with `cdktf`, you can use `cdktf synth` to achieve a similar effect.

In this case, the line between Jsonnet and `cdktf` comes down to the implementation language: whether you want to write
Terraform using Jsonnet, HCL, or a general purpose programming language. This will mostly come down to personal
preference, but Jsonnet has a few notable differentiating factors:

- **Hermeticity**. Jsonnet code is hermetic, meaning that the same JSON output is always generated regardless of the
  runtime environment (e.g., in CI or locally).
- **Imposed limitations**. Jsonnet is not a free-form general purpose language. This means that what you can do in
  Jsonnet is limited (e.g., you can not make a network call to retrieve resource information). Whether you see this as a
  disadvantage or advantage will depend on your specific use case and environment. In some situations, this is a
  desirable factor as you can ensure your code is kept simple with limited points of failure. It also increases the
  safety of consuming third party libraries, since you can be sure it won't be able to do certain things like installing
  and executing a foreign binary.
- **Simple**. Jsonnet is a relatively simple language. The [official
  tutorial](https://jsonnet.org/learning/tutorial.html) covers everything there is about Jsonnet, and can be completed
  in a day at most.

If you are convinced that Jsonnet will work for you, then `tf.libsonnet` should greatly enhance the Terraform writing
experience.
