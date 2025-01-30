# Terralith

A PoC of a tool to manage a Terralith root module.  It is not meant for production use.

Terralith uses modules in the root module to specify different stacks.  A plan can be performed against the entire root module or a list of stacks.  For each stack that is specified, `-target` is used to narrow the plan to the module.  Planning turns refresh and state locking off, enabling multiple plan operations to be executed concurrently.

# Usage

All commands are run in the Terralith root module directory.

To increase the log level add `--log-level=debug` before a sub-command.

## list-stacks

`terralith list-stacks`

Outputs a list of stacks that Terralith has discovered in the root module.

## plan

`terralith plan [-s stack1 [-s stack2 ...]] -o /path/to/plan`

Perform a plan operation and produce a plan file.  Optionally specify one or more stacks to perform the plan on.

## apply

`terralith apply -p path/toplan`

Apply a previously created plan.

# Benefits

The best practices in Terraform and Tofu is to split large infrastructure definitions across multiple root modules.  While this has some benefits, it makes it harder to work, especially with inter-dependencies between root modules.  For example, it is no longer possible to see the plan, in totality, of a change, making it unclear resources are impacted by the change.

Terralith replicates the feel of a multi root module repository design with the benefits of a single root module solution.  Operations can be performed on individual stacks, improving the performance of any individual operation.  Terralith will perform a plan on a specified stack and any impacted dependencies.

Terralith requires a slight refactoring of root modules to be turned into a Terralith, mostly around moving providers around and some state surgery to migrate.

# Speed

A major issue in Terralith's is performing a plan or apply can be time consuming because the entire root module is refreshed and compared.  This makes iterations slow because it takes a long time to perform a plan and with state locking enabled, only one operation can be performed at a time.  Terralith addresses this in three ways:

1. Operations can be performed on individual stacks, reducing the amount of work done in a plan.
2. When planning, state locking is disabled.
3. When planning, state refresh is disabled.

This ensure that a plan does not modify the state and as many plans as possible can be executed concurrently.

# OpenTofu

By default, Terralith uses the `terraform` command.  To use `tofu`, specify `TF_CMD=tofu`.

# Migration

To migrate an existing repository that is broken up into multiple root modules to a Terralith, make the following changes:

1. Move all existing root modules into their own directory if they aren't already.
2. Create a new root module, this will be the Terralith.
3. Move all provider definitions from the old root modules into the new one, refactor them such that credentials are specified per-stack.  This can be spread over many files.
4. Create a `module` block with the name of the stack and specify the `source` as the path to the old root module, and pass the provider in as an alias.
5. Execute `terralith list-stacks` and verify the output matches expectations.
6. Use `terraform state` to move all resources into a single state file.

**Note**: For any stacks which have inter-dependencies, replace the remote data source with module inputs and outputs.

# But isn't -target dangerous?

No!  The `-target` parameter is not dangerous, the problem with it is really that most uses of it are unprincipled, working to get around problems they created.  `terralith` uses `-target` but it uses it in a principled and structured way.  It limits what can be targeted to modules which represent stacks.
