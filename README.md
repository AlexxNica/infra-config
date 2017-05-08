# Fuchsia Continuous Integration Configuration

This repository contains Fuchsia's configuration files for LUCI, the continuous
integration system that we share with the Chromium project.

You can configure LUCI to:

* Run **scheduled jobs** to continuously build and test code in your repo.
* Run a **commit queue** to block commits in Gerrit that would break your build
  or any builds that depend on your code.

## Buildbucket configuration

[Buildbucket](https://chromium.googlesource.com/infra/infra/+/master/appengine/cr-buildbucket/README.md)
is LUCI's build queue. We use Buildbucket with Swarming, which is a system for
centrally scheduling jobs to run on different machines. Swarming is the same
system that the Chromium project uses.

[cr-buildbucket.cfg](services/cr-buildbucket.cfg) defines a set of
**builders** for Fuchsia. A builder basically defines a task to run, and a type
of machine to run it on.

### Anatomy of a builder

Here's one of our builders, as an example:

```
builders {
  category: "Fuchsia"
  name: "fuchsia-x86_64-linux-debug"
  mixins: "fuchsia"
  dimensions: "os:Linux"
  recipe {
    properties: "target:x86-64"
    properties: "build_type:debug"
  }
}
```

This builder has:

* A **category** of similar builders that it belongs to. In this case, it builds
  all of Fuchsia, so it's part of the "Fuchsia" category.
* A unique **name**.
* A set of **dimensions** (in this case just one) specifying what kind of
  machine can handle it.
* Information about what **recipe** to run. A recipe is a Python script that
  uses the
  [recipes framework](https://chromium.googlesource.com/external/github.com/luci/recipes-py/+/master/doc/user_guide.md)
  to define a series of commands to run on the machine. We maintain the Fuchsia
  recipes in the
  [infra/recipes repo](https://fuchsia.googlesource.com/infra/recipes). There
  are two properties here, which will be passed into the recipe when it runs.
  One of them says that the build target is x86_64, and the other one says that
  it's a debug build. It's up to the recipe to interpret these, and you can read
  the recipe code to understand how they get used.

There's also reference to a **builder mixin**, which is a way to reuse
configuration between builders. In this case, we're using the "fuchsia" mixin,
which is defined like this:

```
builder_mixins {
  name: "fuchsia"
  recipe {
    name: "fuchsia"
    properties: "manifest:fuchsia"
  }
}
```

This mixin provides additional information about the recipe: it says which
recipe to use ("fuchsia", which corresponds to a file called
[fuchsia.py](https://fuchsia.googlesource.com/infra/recipes/+/master/recipes/fuchsia.py)
in the recipes repo) and it gives an additional property: it tells the recipe
to use the Jiri manifest named "fuchsia" when it checks out the code.

And finally, this builder is in a **bucket**, which is a grouping of builders
with the same set of ACLs. We have two buckets:

* The **continuous** bucket is for continuously-running post-commit builds.
* The **try** bucket is for testing changes on the commit queue.

This particular builder is in the continuous bucket, which declares a set of
defaults for all of its builders:

```
builder_defaults {
  swarming_tags: "allow_milo:1"
  dimensions: "pool:Fuchsia-try"
  dimensions: "netboot_devices:0"
  recipe {
    repository: "https://fuchsia.googlesource.com/infra/recipes"
    properties: "remote:https://fuchsia.googlesource.com/manifest"
  }
}
```

There are more machine dimensions here, including the machine pool, since the
continuous jobs and try jobs use different pools. There's also the location of
the recipes repos, and the "remote" property for the recipe, which points to the
Fuchsia manifest repo.

### Adding builders for a new project

Since you want to schedule continuous builds and a commit queue, you need to
add the following:

* A builder mixin with all of the common builder parameters for your project.
* A builder in the continuous bucket for each variation that you want to run.
  You probably want at least one each for aarch64 and x86_64.
* A corresponding builder in the try bucket for each variation.
* Any other mixins you need to make the configuration code easy to manage.
  Mixins are allowed to inherit from other mixins, so you can layer them if you
  need to.

## LUCI scheduler configuration

[luci-scheduler.cfg](services/luci-scheduler.cfg) tells LUCI which builds to
run continuously. It consists of multiple `job` declarations, each one of which
refers to a builder using the names defined in the Buildbucket configuration.
This is basically cron for LUCI, and you can use cron expressions in the
`schedule` parameter of each job, as well as expressions like: "with 10m
interval".

## Commit queue configuration

The [repositories](repositories) directory of this repo has a directory for
every Fuchsia repo that has a commit queue. If you want to add a commit queue
for your repo, create a directory here, copy the `refs.cfg` and `cq.cfg` files
from an existing directory, and make the necessary changes.

### `refs.cfg`

This is a magical workaround for legacy issues. It's basically the file that
bootstraps the commit queue configuration. Just make sure that `config_path`
points to your new directory. You also need to make sure this file is picked up
in the global LUCI configuration, which is stored in a Google-internal repo.
Ask someone from the Fuchsia infrastructure team for help with this step.

### `cq.cfg`

This the configuration file for the Chromium Commit Queue, which is a thing that
watches Gerrit for CLs it can build, test, and commit.

We always use the same `cq_status_url` for Fuchsia, and the `git_repo_url` is
specific to the repo. `cq_name` is a unique identifier for the project's CQ,
and it can usually just be the same as the all-lowercase name of the repo.

A LUCI commit queue configuration consists of one or more **verifiers**, which
are steps that must pass before committing is allowed. The basic configuration
for a Fuchsia CQ is a single **try-job verifier**, which means that every commit
must pass a check that involves running a set of builders from the Buildbucket
configuration.

Under `buckets`, we always specify that we are using "luci.fuchsia.try". Add
an entry here for every builder in the try bucket that you want to run on the
CQ for this repo. This could include builders specific to your repo, as well as
anything else you don't want to break when you commit things, such as the main
Fuchsia build.

The `gerrit_cq_ability` section defines permissions for who can use the CQ, and
it's the same for all Fuchsia repos.

## Testing your configuration

Right now the only real way to test your LUCI configuration is to commit a
change and see if it works, then commit a fix if it doesn't. Once your
configuration gets committed, your scheduled builds should start showing up on
the [LUCI Scheduler](https://luci-scheduler.appspot.com/) web interface.

To test the commit queue, create a new commit for your project in Gerrit. The
contents of the commit don't matter, since you will only be doing a dry run.

If you have the right permissions in Gerrit and you are using
[PolyGerrit](https://fuchsia-review.googlesource.com/?polygerrit=1),
you should see a button in the top right that says "CQ Dry Run". Click it, and
soon you should see a comment from the CQ bot, with a link to its progress.
Check to see if it completes all the builds you configured in the above steps.
