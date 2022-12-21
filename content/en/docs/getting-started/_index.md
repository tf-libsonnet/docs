---
title: Getting started
description: Get started using `tf.libsonnet`
no_list: true
weight: 10
---

In this section we cover installing `tf.libsonnet` and getting started with writing your first Terraform Jsonnet code.

## Target Terraform module

In this guide, we will work on using `tf.libsonnet` to generate the equivalent JSON configuration for the following
Terraform code:

```hcl
terraform {
  required_providers {
    null = {
      source = "hashicorp/source"
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

## Installing Jsonnet and tf.libsonnet/core

To get started using `tf.libsonnet`, you need to first install [Jsonnet](https://jsonnet.org/) and
[jsonnet-bundler (pkg manager)](https://github.com/jsonnet-bundler/jsonnet-bundler). Since `tf.libsonnet` is a pure
Jsonnet library for generating Terraform code, we can use the official tools for executing the code.

- [Installing Jsonnet](https://github.com/google/go-jsonnet#installation-instructions)
- [Installing jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler#install)

Once you have `jsonnet` and `jb` installed, you can start adding the `tf.libsonnet` libraries to your project. We will
start with the [core library](https://github.com/tf-libsonnet/core), and later on switch to using the dedicated provider
library.

Run the following to install `tf.libsonnet/core` with `jb`:

```text
jb init
jb install github.com/tf-libsonnet/core@v0.0.2
```

## Adding a null_resource

Now that `tf.libsonnet/core` is installed, let's start building the Terraform module!

The `core` library contains utilities for generating the base Terraform blocks in your document. You can see the list of
exported functions in the [reference docs](https://github.com/tf-libsonnet/core/tree/main/docs). We will start with
generating the `null_resource` block by using
[withResource](https://github.com/tf-libsonnet/core/tree/main/docs#fn-withresource) function.

In your editor, create a file named `main.tf.jsonnet` and add the following:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';

tf.withResource('null_resource', 'this', {})
```

The above code imports the `tf.libsonnet/core` library and uses it to generate a single `null_resource` resource block
in the final document.

You can generate the corresponding Terraform code using the `jsonnet` CLI:

```text
jsonnet -J ./vendor -c -o out/main.tf.json main.tf.jsonnet
```

Here is an explanation of the arguments we are passing to `jsonnet`:

- `-J ./vendor` adds the `vendor` directory created by `jb` to the library path. This ensures that `jsonnet` can find
  `tf.libsonnet` when resolving the `import` calls.
- `-o out/main.tf.json` tells `jsonnet` where to store the resulting JSON file.
- `-c` tells `jsonnet` create the output directories.
- `main.tf.jsonnet` is the file we want to compile with `jsonnet`.

You should see the compiled Terraform code in the `./out/main.tf.json` file, which should look like the following:

```json
{
   "resource": {
      "null_resource": {
         "this": { }
      }
   }
}
```

You can now execute `terraform` against the compiled code. Run the following to try it out!

```text
cd out
terraform init
terraform apply
```

## Adding a variable and binding triggers

Let's augment the `null_resource` with a trigger. The `triggers` attributes allows you to control when the
`null_resource` should be regenerated.

Use the [withVariable](https://github.com/tf-libsonnet/core/tree/main/docs#fn-withvariable) function to add a variable,
and link it to the `triggers` attribute:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';

tf.withVariable('trigger_id')
+ tf.withResource('null_resource', 'this', {
  triggers: {
    trigger_id: '${var.trigger_id}',
  },
})
```

When compiled, the resulting code should look like the following:

```json
{
   "resource": {
      "null_resource": {
         "this": {
            "triggers": {
               "trigger_id": "${var.trigger_id}"
            }
         }
      }
   },
   "variable": {
      "trigger_id": { }
   }
}
```

In this way, you can use the `attrs` parameter of the `withResource` function to set the different attributes of the
resulting Terraform block.

## Adding the required_providers block

The above code works as is, but in production you will want to ensure you are version locking your providers. You can do
this by adding the [required_providers](https://developer.hashicorp.com/terraform/language/providers/requirements)
Terraform block. In `tf.libsonnet`, this block can be added with the
[withProvider](https://github.com/tf-libsonnet/core/tree/main/docs#fn-withprovider) function.

Update your `main.tf.jsonnet` file with the following:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';

tf.withProvider('null', {}, src='hashicorp/null', version='~>3.0')
+ tf.withVariable('trigger_id')
+ tf.withResource('null_resource', 'this', {
  triggers: {
    trigger_id: '${var.trigger_id}',
  },
})
```

Recompile the code, and inspect the resulting Terraform JSON. It should now include the `required_providers` block to
version lock the `null` provider:

```json
{
   "provider": {
      "null": [
         { }
      ]
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
   "terraform": {
      "required_providers": {
         "null": {
            "source": "hashicorp/null",
            "version": "~>3.0"
         }
      }
   },
   "variable": {
      "trigger_id": { }
   }
}
```

## Adding the output reference

Let's complete the module by outputing the `null_resource` ID. To do this, we will take advantage of the self reference
generator injected by the `withResource` function. As indicated in the [withResource function
docs](https://github.com/tf-libsonnet/core/tree/main/docs#fn-withresource), every resource injects a self reference
to the `_ref` attribute.

To be able to use the self reference links, you need to bind the resource to a `local` reference. Update your code with
the following:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';

local o =
  tf.withProvider('null', {}, src='hashicorp/null', version='~>3.0')
  + tf.withVariable('trigger_id')
  + tf.withResource('null_resource', 'this', {
    triggers: {
      trigger_id: '${var.trigger_id}',
    },
  })
  + tf.withOutput(
    'this_id',
    o._ref.null_resource.this.get('id'),
  );

// Don't forget to declare the local var `o` as the final object for the document!
o
```

Note the `o._ref.null_resource.this.get('id')`. When compiled, this generates the Terraform interpolation to reference
the `id` field of the `this` instance of the `null_resource` resource (`${null_resource.this.id}`). You can of course
replace that with the raw string, but using the `_ref` reference ensures that you are referencing resources that exist
in the document. In essence, it provides a compile time assertion as opposed to run time assertion (e.g., you can
validate before the code is generated, vs checking during a `terraform validate` call).

This will generate the final JSON we want for our module:

```json
{
   "output": {
      "this_id": {
         "value": "${null_resource.this.id}"
      }
   },
   "provider": {
      "null": [
         { }
      ]
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
   "terraform": {
      "required_providers": {
         "null": {
            "source": "hashicorp/null",
            "version": "~>3.0"
         }
      }
   },
   "variable": {
      "trigger_id": { }
   }
}
```

## Using provider specific libraries

Using the `core` library allows you to generate arbitrary Terraform code, but with real world use cases, it is better to
have some type safety. Jsonnet is not a statically typed language so you won't get full static type safety like you do
with TypeScript, but you can get some limited form of type safety by using the provider specific libraries in
`tf.libsonnet`. These libraries export every resource and data source that is supported by the provider as Jsonnet
functions. Using these libraries can give you compile time guarantees for the references and attributes.

For example, if you had misspelled `triggers` in the attribute object, `jsonnet` will compile the code down to
JSON, but `terraform` will not be happy with it.

Refer to the [Supported Providers](/docs/supported-providers) page for the list of officially maintained provider libraries.

Let's shift this check left by using the `tf.libsonnet/hashicorp-null` library to generate the resource. Update the
code to reference the `null` provider library:

```text
local tf = import 'github.com/tf-libsonnet/core/main.libsonnet';
// NOTE: ideally we can bind `null`, but `null` is a reserved word so we use `tfnull` as an alternative.
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

Before you can compile the code, you will need to make sure the library is available to Jsonnet. Install the library with `jb`:

```text
jb install github.com/tf-libsonnet/hashicorp-null@v0.0.3
```

Finally, run `jsonnet` to generate the compiled code and verify your results.

Note that this is where the true strengths of the compiler pattern emerges. This switch to `tf.libsonnet/hashicorp-null`
is considered a refactor of your Jsonnet code. You are modifying the Jsonnet code without changing the resulting
Terraform. This means that you can be confident that you didn't change any behavior as long as the generated code is
equivalent: you don't need to run `terraform validate` or `terraform plan` to check! To drive this, you can output the
code to a different directory and then run a `diff` to verify there are no changes. Or in real world scenarios, you can
rely on `git` and GitOps, where you only commit the resulting Terraform files if there are any changes.

## Where to go from here

Learn more about the Jsonnet language:
- [Introduction to Jsonnet](https://jsonnet.org/learning/tutorial.html)
- [Standard Library](https://jsonnet.org/ref/stdlib.html)
- [Language Design](https://jsonnet.org/articles/design.html)

Get help on the [tf.libsonnet GitHub Discussion](https://github.com/orgs/tf-libsonnet/discussions).
