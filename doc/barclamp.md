# Barclamp development guide

This document is an introduction to Barclamp development. It will guide you
through all the steps neccessary to create a minimal (without UI elements)
Barclamp for Crowbar.

## Prerequisites

We will assume some things to already be present. Namely, the following:

* A Github fork of the appropriate Barclamp collection. For instance, you will
  find OpenStack related barclamps in the
  [crowbar-openstack](https://www.github.com/crowbar/crowbar-openstack)
  repository and high availability related Barclamps in the
  [crowbar-ha](https://github.com/crowbar/crowbar-ha) repository. For the sake
  of readability we will assume you are developing an OpenStack related and
  forked [crowbar-openstack](https://www.github.com/crowbar/crowbar-openstack)
  throughout the rest of this document.

* A Crowbar admin node with the Crowbar version you are developing against
  (for new barclamps this will be `master`) already installed and running. This
  should be as up-to-date as possible to reduce the chance of merge conflicts
  happening.

* A set of machines for your Crowbar admin node to orchestrate. In the case of
  SUSE OpenStack Cloud this would be a controller and one compute node at a
  minimum.

Providing these is out of scope for this document. We assume you are able to
fork the appropriate Barclamp collection. The machines can be provided through
manual installation or by using an automated setup tool such as
[mkcloud](https://github.com/SUSE-Cloud/automation/blob/master/scripts/mkcloud).

Optionally (but strongly recommended) you can also create a backup of your
Crowbar admin node. You are probably going to break it a few times in the
course of testing your new barclamp so it is a good idea to have a backup you
can quickly restore rather than rebuild your entire test environment. If you
are using `mkcloud` you can create and restore a LVM snapshot of your admin
node using its `createadminsnapshot` and `restoreadminfromsnapshot` steps,
respectively.

## Development Environment

Most of your development will take place on your Crowbar node (log in as root).
You will find the currently running Crowbar code (both Rails application
components and chef recipes) in the `/opt/dell` directory. We will provide
detailed locations for all important components of a Barclamp in the _Basic
Implementation_ section below.

In order to track changes you make we recommend you keep a checkout of your
`crowbar-openstack` fork on the Crowbar node and apply the changes you make in
there

### Preparations for Crowbar Node

These are recommendations, but we'll assume you followed them for the remainder
of this document since this shortens the commands in our examples. First of
all, clone your `crowbar-openstack` fork to `/root/crowbar-openstack` on your
Crowbar node and switch to your development branch.

Next, create the following /root/.bashrc with a few useful aliases and
environment variables (substitute with appropriate values where so indicated by
comments):

```
# Replace "Crowbar Committer" by your name as it should appear in commit
# messages.
C_NAME="Crowbar Committer"

# Replace "email@address.invalid" by your email address as it should appear in
# commit messages.
C_EMAIL="email@address.invalid"

# Replace "6cefbd80254141a63f4c86df195cd930ac7eb10b" by your first commit's ID.
export COMMIT="6cefbd80254141a63f4c86df195cd930ac7eb10b"

# Insert a space delimited list of the Barclamps you are working on here.
export BARCLAMPS="nova magnum"

export GIT_AUTHOR_NAME="$C_NAME"
export GIT_AUTHOR_EMAIL="$C_EMAIL"
export GIT_COMMITTER_NAME="$C_NAME"
export GIT_COMMITTER_EMAIL="#C_EMAIL"

alias Cpatch="(cd ~/crowbar-openstack; git show | patch -p1 --merge -d /opt/dell)"

# Full patch: applies your fork completely against /opt/crowbar
function Fpatch()
  {
  cd ~/crowbar-openstack
  git diff "$COMMIT^" HEAD | patch --merge -p1 -d /opt/dell
  for f in /opt/dell/*.yml
    do
    ln -s $f /opt/dell/crowbar_framework/barclamps 2>&1 > /dev/null
    done
  chmod 755 /opt/dell/bin/*
  }

sync_crowbar()
  {
  barclamp_install.rb /opt/dell/crowbar_framework/barclamps/ &&
  for barclamp in $BARCLAMPS
    do
    knife cookbook upload -o /opt/dell/chef/cookbooks/ monasca
    done
  systemctl restart crowbar
  }
```

We will make heavy use of the `Cpatch`, `Fpatch` and `sync_crowbar` commands
thus defined in the following sections. A short explanation of the three:

* `Fpatch` will apply your whole fork against `/opt/dell` using `patch(1)`. You
  use this command to apply the current state of your fork to a freshly
  installed Crowbar node. This command may cause merge conflicts. If this
  happens, resolve them before doing anything else.

* `Cpatch` will apply the latest commit in `/root/crowbar-openstack` to
  `/opt/dell` (use this after each new commit). This command may cause merge
  conflicts. If this happens, resolve them before doing anything else.

* `sync-crowbar` will make any updated state in `/opt/dell` appear in the
  running Crowbar instance. Run this after every change, no matter how minor,
  in `/opt/dell`.

Using patches against `/opt/dell` may seem a bit over-the-top at first since
one could simply copy files back and forth, but it prevents breakage in cases
where `/opt/dell` is out of sync with the `crowbar-openstack` repository's
master branch or your own fork of `crowbar-openstack` is out of sync with
upstream: in the former case copying files from your fork to `/opt/dell` may
cause an inconsistent state and errors entirely unrelated to your change. In
the latter case copying files from `/opt/dell` to your fork may re-introduce
old code that is no longer present upstream. Both should not be an issue for
new barclamps, but it is better to get used to the `patch(1)` based workflow
right away since these problems do become a problem when changing an existing
barclamp later and thus the `patch(1)` based workflow becomes neccessary.

### Development Workflow

In this section we will show a development workflow we consider useful. This is
not the end-all-be-all of Barclamp development workflows, but it should be
useful for new developers.

## Development Phases

The Development workflow essentially consists of 3 phases: 

1. Basic Implementation: In this phase you will write a scaffold for the
   Barclamp's chef code, define roles and parameters and create UI code for
   parametrizing your Barclamp. Note that no testing needs to take place during
   this phase (and doesn't make much sense either). This phase produces a rough
   draft that may have any number of errors but will already define your
   barclamp's basic architecture.

2. Local Testing: in this phase you will debug and fix the rough draft from the
   first phase until the UI side of the barclamp works well and parametrizes
   the barclamp as expected. As a second step you will fill your chef recipe
   scaffold with life and develop your chef recipes to the point where they
   correctly deploy what they are supposed to deploy.

3. Pull Request Testing: once you are finished with phase 2 to the point where
   the barclamp performs to your own satisfaction you submit a pull request for
   your barclamp against Crowbar's `master` branch. Now Crowbar's CI tests will
   run using the code from your pull request and Crowbar maintainers will
   review it. During this phase you may again have to fix a number of things
   and may well need to return to phase 2 for local testing of the changes you
   make (it is not recommended to rely on CI tests by themselves for extensive
   changes).

Below we will describe these phases in detail.

_Note: unless otherwise indicated, all paths from here on out are relative to
your_ `crowbar-openstack` _fork's root directory._

### Basic implementation

In this phase you will do a scaffold implementation of your barclamp. This
means that you will mostly focus on the Crowbar UI side of things and how it
ties into your barclamp's chef cookbook. This means that on the chef side
you'll only implement the structure of your chef cookbook, i.e. define the
parameters it takes and the roles it defines for the Crowbar UI to use.

You should also create recipes and configuration templates in your cookbook
already, but you only need to flesh these out as far as you need them to be
clear about the parameters you'll need Crowbar to supply.

#### Chef Cookbook

Location: `chef/cookbooks/<your_barclamp_name>`

The best point to start barclamp development is the
[chef cookbook](https://docs.chef.io/cookbooks.html). Each barclamp contains
one of these. The chef cookbook is your barclamp's core component. It contains
the code that will do the grunt work of deploying the services, configuration
files and runtime state handled by your barclamp. 

#### Recipes

Location: `chef/cookbooks/<your_barclamp_name>/recipes`

A chef cookbook consists of
[https://docs.chef.io/recipes.html](https://docs.chef.io/recipes.html). These
recipes are written in Ruby (hence they must have a `.rb` extension) and are
the smallest organization unit of a chef cookbook. They consist of
[resource](https://docs.chef.io/resource.html) definitions that define various
aspects of system state, such as a configuration file's content, packages to
install, or a service to enable and start. In this step the recipes do not need
to contain any code, yet. You primarily need to know which recipes to define
and what each recipe's purpose is at this stage.

You are free to organize your recipes as you see fit. There are no
hard-and-fast rules for this since they mostly depend on the application your
barclamp deploys:

For instance, if your application (like most OpenStack components) consists of
an API service and a RabbitMQ driven backend service, you might want to split
these into a `api.rb` and `backend.rb` service. Likewise, if you need an
agent to run on compute nodes, this might warrant a dedicated `agent.rb`
recipe. If you want to use Crowbar's high availability features, it might also
make sense to have a dedicated `ha.rb` recipe for users deploying it in a
highly available manner.

On a more general note, feel free to take a look at the other cookbooks'
recipes in `chef/cookbooks/` for inspiration. We have barclamps for a range of
OpenStack services already and one of these might just match your own
application's architecture perfectly.

#### Roles

Locations:

* Role recipes: `chef/cookbooks/<your_barclamp_name>/recipes`
* Role definitions: `chef/roles`

Once you've got a set of chef recipes, you'll need to organize them into roles.
A role aggregates one or more chef recipes and deploys all of its component
chef recipes. Crowbar can assign roles to one or more nodes (subject to
constraints defined in the Crowbar application). In the HA case it can assign
roles to one or more clusters of roles.

Roles are defined in two places. First, you create a _role recipe_ in the
`chef/cookbooks/<your_barclamp_name>/recipes` directory. Our naming convention
for role recipes is as follows:

```
role_<barclamp name>_<role description>
```

`<role description>` may contain letters, numbers and underscore (`_`)
characters.

This role recipe contains one or more `include_recipe` statements to pull in
component recipes and a validity check that asks the Crowbar application
whether the role is valid for the current node. Here's an example from the
Barbican barclamp (`cookbooks/barbican/recipes/role_barbican_controller.rb`):

```
if CrowbarRoleRecipe.node_state_valid_for_role?(node, "barbican", "barbican-controller")
  include_recipe "barbican::api"
  include_recipe "barbican::common"
  include_recipe "barbican::worker"
  include_recipe "barbican::keystone-listener"
  include_recipe "barbican::ha"
end
```

Note how the recipes are qualified with the cookbook name ("barbican") in this
case and omit the `.rb` extension from their actual file name.

Once all role recipes are created, you can create the corresponding role
definitions in the `chef/roles` directory. Role definitions are snippets of
ruby code and must have a `.rb` definitions. Our naming convention for role
definitions is as follows:

```
<barclamp name>-<role description>
```

where `<role description>` is the role description from the role recipe with
underscores (`_`) substituted by dashes (`-`). This will be the role's
canonical name from this point onward.

A role definition looks as follows (example from `roles/barbican-controller.rb`):

```
name "barbican-controller"
description "Barbican Controller Role"
run_list("recipe[barbican::role_barbican_controller]")
default_attributes
override_attributes
```

Most of this is boilerplate with two notable exceptions:

1. `name` defines the role's canonical name as described above
2. `run_list` contains a reference to the role's underlying role recipe.

Create role definitions for all of your roles now and proceed to the next
section.

#### Data Bag

#### Crowbar Application Components

### Local Testing

#### Create Proposal

#### Apply Proposal

#### Implement and Debug Chef Recipes

### Pull Request Testing

#### Fix Hound Violations

#### Mkcloud Gating

#### Reviewer approval

#### 
