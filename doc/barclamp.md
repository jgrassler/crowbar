# Barclamp development guide

This document is an introduction to Barclamp development. It will guide you
through all the steps neccessary to create a minimal (without UI elements)
Barclamp for Crowbar. It will then guide you through all the testing, CI and
review steps until the point where it gets merged into Crowbar.

## Terms and definitions

We will assume throughout this text that your barclamp is will be named
`mybarclamp`. Please substitute your actual barclamp's name for this
placeholder when you create a barclamp. The convention for OpenStack barclamps
is to use the OpenStack component's name, e.g. `nova`.

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

Optionally (but strongly recommended) you can also

* Create a backup of your Crowbar admin node: You are probably going to break
  it a few times in the course of testing your new barclamp so it is a good
  idea to have a backup you can quickly restore rather than rebuild your entire
  test environment. If you are using `mkcloud` you can create and restore a LVM
  snapshot of your admin node using its `createadminsnapshot` and
  `restoreadminfromsnapshot` steps, respectively.

* Create an additional OpenStack controller or compute node as a scratch node:
  this will allow you to deploy only your own barclamp's controller side
  components to this node while all other OpenStack services reside on the main
  controller. This greatly shortens the time a chef run for your barclamp's
  controller side components takes and thus speeds up development. Make a
  backup of this node as well to guard against breakage and allow for
  clean-slate deployment of your barclamp's controller side components.

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
    knife cookbook upload -o /opt/dell/chef/cookbooks/ mybarclamp
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

## Barclamp Component Overview

A barclamp consists of two main components:

* A [chef cookbook](https://docs.chef.io/cookbooks.html) which takes care of
  actually deploying your service.
* Some glue code to tie your chef cookbook into the Crowbar Web UI. This glue
  code will parametrize your chef cookbook and apply its component roles to the
  nodes it orchestrates.

The subsequent sections will describe these two components and the ingredients
that go into them into detail. They start of with an overview of a barclamp's
development phases as we know them from experience and take you through all
aspects of barclamp implementation in the order you are likely to encounter
them. They are meant to be read in parallel to developing your own barclamp.

## Development Phases

In this section we will show a development workflow we consider useful. This is
not the end-all-be-all of Barclamp development workflows, but it should be
useful for new developers.

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

Throughout this phase you can work on your local machine or on your development
Crowbar admin node, as you prefer, since the testing you can do in this phase is
limited to basic ruby syntax checks using `ruby -c`. Once you are finished with
this phase development will have to move to your Crowbar admin node, though.

#### Chef Cookbook

Location: `chef/cookbooks/`

The best point to start barclamp development is the
chef cookbook. Each barclamp contains
one of these. The chef cookbook is your barclamp's core component. It contains
the code that will do the grunt work of deploying the services, configuration
files and runtime state handled by your barclamp. As a first step, create
`chef/cookbooks/` now.

#### Cookbook metadata

Locations:

* `chef/cookbooks/mybarclamp/metadata.rb`
* `chef/cookbooks/mybarclamp/README.md`

Before we begin with the cookbook proper we will need to take care of some
boilerplate files. The first of these is `metadata.rb`. It mostly contains
various descriptive information about your barclamps. Here's an example from
the Barbican barclamp (`chef/cookbooks/barbican/metadata.rb`):

```
name "barbican"
maintainer "Crowbar project"
maintainer_email "crowbar@googlegroups.com"
license "Apache 2.0"
description "Installs/Configures Barbican"
long_description IO.read(File.join(File.dirname(__FILE__), "README.md"))
version "0.1"

depends "apache2"
depends "database"
depends "keystone"
depends "crowbar-openstack"
depends "crowbar-pacemaker"
depends "utils"
```

The contents of these fields are mostly free-form, hence you can copy the file
from another barclamp and simply adapt the `name` and `description` fields. The
one exception to that rule are the `depends` statements: these define which
other barclamps your own barclamp depends on because it uses their cookbooks'
contents. If, for instance, you use the Keystone barclamp's
`KeystoneHelper.keystone_settings()` method (which you should for OpenStack
barclamps), you'd add a `depends "keystone"` statement here.

You may already have noticed the second piece of boilerplate: the README.md
file referenced by the `long_description` statement. This goes one goes into
`chef/cookbooks/` and describes your Chef cookbook as
verbosely or tersely as you like (please put at least one complete sentence in
there).

#### Recipes

Location: `chef/cookbooks/mybarclamp/recipes/*.rb`

[Recipes](https://docs.chef.io/recipes.html) make up the bulk of your chef
cookbook's payload. These recipes are written in Ruby (hence they must have a
`.rb` extension) and are the smallest organization unit of a chef cookbook.
They consist of [resource](https://docs.chef.io/resource.html) definitions that
define various aspects of system state, such as a configuration file's content,
packages to install, or a service to enable and start. In this step the recipes
do not need to contain any code, yet. You primarily need to know which recipes
to define and what each recipe's purpose is at this stage.

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

* Role recipes: `chef/cookbooks/mybarclamp/recipes`
* Role definitions: `chef/roles/role_mybarclamp_*.rb`

Once you've got a set of chef recipes, you'll need to organize them into roles.
A role aggregates one or more chef recipes and deploys all of its component
chef recipes. Crowbar can assign roles to one or more nodes (subject to
constraints defined in the Crowbar application). In the HA case it can assign
roles to one or more clusters of roles.

Roles are defined in two places. First, you create a _role recipe_ in the
`chef/cookbooks/<your_barclamp_name>/recipes` directory. Our naming convention
for role recipes is as follows:

```
role_<barclamp name>_<role description>.rb
```

`<role description>` may contain letters, numbers and underscore (`_`)
characters. Assuming you've got two separate role recipes, one for the compute
nodes and one for OpenStack controller nodes, you might end up with something
like this:

* `chef/cookbooks/mybarclamp/role_mybarclamp_compute.rb`
* `chef/cookbooks/mybarclamp/role_mybarclamp_controller.rb`

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

Most of this is boilerplate, with two notable exceptions:

1. `name` defines the role's canonical name as described above
2. `run_list` contains a reference to the role's underlying role recipe.

Create role definitions for all of your roles now and proceed to the next
section.

#### Data Bag

Locations:

* Data bag schema: `chef/data_bags/crowbar/template-mybarclamp.schema`
* Default data bag: `chef/data_bags/crowbar/template-mybarclamp.json`
* Migrations: `chef/data_bags/crowbar/migrate/mybarclamp/*.rb`

Now that you have your recipes and roles defined you should also know what
configuration settings are static (e.g. best practice settings you want to
hardwire as enabled or disabled) and what needs to go into your chef cookbooks
as parameters that can vary for each local setup (e.g. user names and passwords
or the OpenStack identity URL). Everything you want to be configurable will
need to go into the `data bag`.

For a newly created Barclamp the data bag consists of a _data bag schema_
(`chef/data_bags/crowbar/template-mybarclamp.schema`) and a
_default data bag_ (`chef/data_bags/crowbar/template-mybarclamp.json`). Both
are in JSON format. The _data bag schema_ describes how your data bag is
structured and imposes constraints such as data type and permissible values on
individual fields (much like an XML schema). It is used to validate your
barclamp's parameters, especially when the user edits its parameters directly
as JSON, either in the Web UI's raw mode or through the Crowbar command line
client.

The _default data bag_ contains default settings for these parameters where
defaults can reasonably be provided. There are some special cases such as user
names and passwords that are initially left blank and will be filled in by the
Crowbar web UI when the barclamp is activated, so feel free to leave fields
blank if you plan on adding code to the Crowbar web UI to populate them or make
use of existing web UI.

When fields are added to or removed from an existing barclamp's data bag, you
will need to add another component to the data bag: a _migration_. This is
essentially a database migration for the Crowbar web UI since data bags map
directly to a table in the Crowbar web UI's database. To keep that database
in sync with the data bag you will need to provide migrations to update that
database table if you make changes to an existing barclamp's data bag.

In the following sections we will explain the data bag's components in detail.
We will start with the default data bag rather than the data bag schema because
it is easier to fill that with content first, and then produce a schema to match.

##### Default Data Bag

* Location: `chef/data_bags/crowbar/template-mybarclamp.json`

First of all, copy an existing default data bag to
`chef/data_bags/crowbar/template-mybarclamp.json`. You will modify
this data bag to contain your own barclamp's parameters. This is easier and far
less error prone than writing a default data bag from scratch. Throughout this
section I will assume you used the
[Barbican barclamp's default data bag](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json)
and link to it when we discuss the changes you need to make.

First of all, replace all occurrences of `barbican` by `` and adjust
the top-level `description field`. Your default data bag's first few lines
should now look something like this:

```
{
  "id": "template-",
  "description": "The Foobar as a Service Component of OpenStack",
  "attributes": {
      "" : {
```

The corresponding section of the Barbican default data bag begins
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L1) and ends
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L29).

With these preliminaries out of the way you can now add all the parameters you
need under `attributes[""]` and remove existing parameters left over
from the default data bag you copied. You should leave the following fields in
place, since the Crowbar UI will automatically populate them for barclamps in
`crowbar-openstack` and/or because some chef side helper functions will use
them:

* `db`
* `database_instance`
* `keystone_instance`
* `rabbitmq_instance`
* `user`
* `group`
* `service_password`
* `service_user`

When you are done `attributes[""]` should look something like this:

```
     "" : {
        "api" : {
           "bind_host" : "*",
           "bind_port" : 12345,
           "logfile" : "/var/log/myservice/myservice-api.log",
        },
        "backend" : {
            db_plugin = "postgres",
            enable_foo = true,
            foobar_threshhold = 255
        },
        "db" : {
           "database" : "myservice",
           "password" : "",
           "user" : "myservice"
        },
        "debug" : false,
        "group" : "",
        "user" : "",
        "database_instance": "none",
        "keystone_instance": "none",
        "rabbitmq_instance": "none",
        "service_password": "none",
        "service_user": "myservice"
     }
```

The corresponding section of the Barbican default data bag begins
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L5) and ends
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L29).

Once you have added your parameters you will need to adjust
`deployment[""]`. This section provides meta data about your chef
cookbook to the Crowbar web UI. The following fields need to be changed in
there:

* `schema-revision`: This field contains the data bag's schema revision. For a
   new barclamp it starts with the `crowbar-core` `master` branch's current
   base revision (at the time of this writing this was `100` but it increases
   with every new stable Crowbar release. Please ask a Crowbar developer about
   the current value). Whenever the data bag is changed, this revision is
   incremented by `1` and a corresponding migration is added to the
   `chef/data_bags/crowbar/migrate/mybarclamp/` directory. Since you are
   writing a new barclamp, set this to  the current `master` branch's base
   revision.

* `element_states`: this field contains a mapping from your chef roles to a
  list of valid states for each role. Typically these states are `readying`,
  `ready` and `applying` for every role, so just use these values for every
  role's list.

* `element_order`: this field is a list of single-element lists and defines the
  order in which your barclamp's roles should be applied. Each single-element
  list contains a role name. Whichever role occurs first in this list must have
  been applied successfully to all nodes it is assigned to before the roles
  further down are applied to their respective nodes. In other words, this
  controls global execution order of your roles across all nodes orchestrated
  by Crowbar. All of your roles should be mentioned here.

* `element_run_list_order`: this field governs the priority of your roles in
  relation to the other roles assigned to any given node your own role is
  assigned to. In other words, this governs local execution order of all roles
  (i.e. not just yours) assigned to a given node. All of your roles should be
  assigned a priority here.

When you are done `deployment[""]` should look something like this
(this example assumes you have defined a `mybarclamp-compute` and a
`-controller` role):

```
  "deployment": {
    "" : {
      "crowbar-revision": 0,
      "crowbar-applied": false,
      "schema-revision": 100,
      "element_states": {
        "mybarclamp-controller": [ "readying", "ready", "applying" ],
        "mybarclamp-compute": [ "readying", "ready", "applying" ]
      },
      "elements": {},
      "element_order": [
        [ "mybarclamp-controller" ],
        [ "mybarclamp-compute" ]
      ],
      "element_run_list_order": {
        "mybarclamp-controller": 120,
        mybarclamp"-compute": 120
      },
      "config": {
        "environment": "mybarclamp-base-config",
        "mode": "full",
        "transitions": false,
        "transition_list": []
      }
    }
  }
```

The corresponding section of the Barbican default data bag begins
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L31) and ends
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.json#L53).

##### Data bag schema

* Location: `chef/data_bags/crowbar/template-mybarclamp.schema`

With your default data bag finished you can now use the attributes and their
data types from the default data bag as a guide for definining its schema. The
schema holds constraints for every attribute in the default data bag, plus
for optional attributes that may not be set in the default data bag but can
nonetheless be added by the user.

The schema contains a hierarchy of dictionaries that mirror the data structure
in `template-mybarclamp.json`. The keys in this hierarchy are the attribute
names from `template-mybarclamp.json` and the values are dictionaries of
constraints for the attribute in question. Such a dictionary of constraints can
have the following fields (these are only the most common ones):

* `required`: a boolean value indicating whether this attribute must be present
  (`true`) or not (`false`).

* `type`: the attribute's data type. Valid values are `str`, `bool`, `int`,
  `map` (key/value pairs) and `sequence` (lists of values).

* `pattern` [only for attributes of type `str`]: a regular expression the attribute
  must match.

* `mapping` and `sequence` [only for attributes of type `map` and `sequence`,
  respectively]: these contain lists or dictionaries that can be used to impose
  constraints on a list or dictionary value's contents. This is mainly used to
  cover the whole tree of nested data dictionaries in the data bag, but it can
  also be used to impose restrictions on an individual list or dictionary
  attribute at the lowest level.

Armed with this knowledge we can now create a schema that matches the default
data bag from the previous section. Similar to the previous section we will
start out with a copy of the Barbican barclamp's schema
(`template-barbican.schema`), which we'll copy to `template-mybarclamp.json`.

Again we will first globally search and replace `barbican` by `` and
then go through it section by section, starting with the top-level fields. The
only thing that needs adjusting at the top level is the pattern constraint for
the `id` field, which our global search and replace will already have taken
care of, leaving us with a top-level section that should look something like this:

```
{
  "required": true,
  "type": "map",
  "mapping": {
    "id": { "type": "str", "required": true, "pattern": "/^-|^template-mybarclamp$/" },
    "description": { "required": true, "type": "str" },
    "attributes": {
      "required": true,
      "type": "map",
      "mapping": {
        "": {
```

The corresponding section from the Barbican data bag schema begins
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.schema#L1) and ends
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/chef/data_bags/crowbar/template-barbican.schema#L11).

Next, we will need to adjust the constraints for `attributes[""]`. The Crowbar
web UI is a Ruby on Rails application with a MVC (Model-View-Controller)
architecture so you will need to create the MVC architecture's three
components for your barclamp plus some glue and metadata. This section will
take you through all of these steps.

##### Barclamp Metadata

Location: `mybarclamp.yml`

Much like the chef cookbook, the Crowbar application needs some metadata about
your barclamp. This is a YAML formatted file residing in the top level
directory. It's filename consists of your barclamp's name plus the extension
`.yml`. Below you will find a commented example:

```
barclamp:
  # Your barclamp's name as it appears in file names.
  name: 'mybarclamp'
  # Your barclamp's name as it will be displayed in the Crowbar web UI.
  display: 'Mybarclamp'
  # Here we usually have the service name and a short description of the
  # Service the barclamp deploys for OpenStack barclamps.
  description: 'OpenStack Myservice: Provides Foo and Bar'
  # Just leave this as it is.
  version: 0
  # This is a repeat of the requirements list from the chef cookbook.
  requires:
    - '@crowbar'
    - 'database'
    - 'rabbitmq'
    - 'keystone'
  # This denotes the Barclamp collection this barclamp belongs to.
  member:
    - 'openstack'

crowbar:
  # The barclamp's priority in the crowbar UI (lower number means higher priority)
  run_order: 98
  # The barclamp's priority in the ordering of chef cookbooks (lower number
  # means higher priority)
  chef_order: 98

  # FIXME: The following appear to be unused. They are neither loaded in
  # catalog() in barclamp_mgmt_lib.rb, nor do they have a corresponding method
  # in the BarclampCatalog class). You might be able to omit them.
  order: 98
  proposal_schema_version: 3
  layout: 1
```

##### Command Line Executable

Location: `bin/crowbar_mybarclamp`

This executable is used by the crowbar command line client and is mostly a
stub. Just copy this from another barclamp and change the `@barclamp` variable
to your barclamp's name, which will leave you with something like this for our
example barclamp:

```
require File.join(File.expand_path(File.dirname(__FILE__)), "barclamp_lib")
@barclamp = "mybarclamp"
@timeout = 3600

main
```
##### Minimal Edit View

Locations:

* `crowbar_framework/app/views/barclamp/mybarclamp/_edit_attributes.html.haml`
* `crowbar_framework/config/locales/mybarclamp/en.yml`

Since we are building a minimal barclamp without much in the way of UI
components, we can stick to the bare minimum for the "View" part of MVC. We
only need its components to be present without much content.

First of all we create the directories

* `crowbar_framework/app/views/barclamp/mybarclamp`
* and `crowbar_framework/config/locales/mybarclamp`.

Next, we create the basic view in
`crowbar_framework/app/views/barclamp/mybarclamp/_edit_attributes.html.haml`.
Since all we need is an edit field for raw parameters in JSON format, we can
get away with only having the following snippet of code in there:

```
= attributes_for @proposal do
  .panel-sub
      = header show_raw_deployment?, true
```

The corresponding file from the Barbican barclamp is
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/views/barclamp/barbican/_edit_attributes.html.haml).


What remains for the view is a minimum English language locale in
"crowbar_framework/config/locales/mybarclamp/en.yml". Since we do not display
any form fields this can be fairly short. To give you an idea of how an actual
form field's description would look we included one in our example below (note
that this would require an entry in
`crowbar_framework/app/views/barclamp/mybarclamp/_edit_attributes.html.haml` to
actually show up):

```
en:
  barclamp:
    mybarclamp:
      edit_attributes:
        api_header: 'API Settings'
        api:
          bind_host: 'Address to listen on'

```

The corresponding file from the Barbican barclamp is
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/config/locales/barbican/en.yml).
It contains field descriptions for a bunch of fields already, since this will
make it easier to add UI elements later. Feel free to prepare your own barclamp
in a similar manner.

##### Minimal Model Class

Location: `crowbar_framework/app/models/mybarclamp_service.rb`

With the view in place we need a model class to pre-process the proposal
information and enforce constraints that cannot be expressed using the
declarative means available in `template-mybarclamp.schema`, i.e. constraints
that need program logic to check. Just like in the data bag sections we will
again use the Barbican barclamp's proposal as an example and modify it bit by
bit, so as a first step, copy `crowbar_framework/app/models/barbican_service.rb`
to `crowbar_framework/app/models/mybarclamp_service.rb` and globally replace
`barbican` by `mybarclamp` and `Barbican` by `Mybarclamp`, respectively.

This will leave us with with something like this in the first few lines:

```
class MybarclampService < PacemakerServiceObject
  def initialize(thelogger)
    @bc_name = "mybarclamp"
    @logger = thelogger
  end
```

The corresponding lines from the Barbican barclamp's controller begin
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L17) and end
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L21).

The next thing we need to change is the `role_constraints` method. This method
tells Crowbar on how many nodes the service can be deployed, what types of node
are valid and on which platform it may run. All the roles you defined in the
chef sections need to be accounted for by this method. This means that we will
need to add an entry for the `mybarclamp-compute` role. Below you will see a
commented example:

```
    def role_constraints
      {
        "mybarclamp-controller" => {
          "unique" => false,   # Set to `true` if this can only be deployed to one node.
          "count" => 3,        # Number of allowed instances ('-1' means infinite)
          "cluster" => true,   # Whether this role can be clustered (important for HA later on).
          "admin" => false,    # Whether this role can be assigned to the Crowbar admin node.
          "exclude_platform" => {
            "suse" => "< 12.1",  # Do not deploy on SUSE version 12.1 and lower.
            "windows" => "/.*/"  # Do not deploy on any Windows platforms.
          }
        },
         "mybarclamp-compute" => {
          "unique" => false,
          "count" => -1,   # Can be deployed to an arbitrary number of Compute nodes
          "cluster" => true,
          "admin" => false,
          "exclude_platform" => {
            "suse" => "< 12.1",
            "windows" => "/.*/"
          }
        }
      end
    end
```
The corresponding lines from the Barbican barclamp's controller begin
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L29) and end
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L43).

Depending on your barclamp's dependencies you may need to adjust the list of
dependencies returned by the `proposal_dependencies()` method. Since `mybarclamp`
has the same dependencies as the Barbican barclamp, that is not neccessary in
our case. You will find the Barbican barclamp's relevant line
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L47).

Now we come to the method where we'll usually have most of our changes:
`create_proposal()`. This method creates the run-time proposal in Crowbar's
data base that contains the parameters as they will be passed to your chef
recipe. It has two main purposes:

1. Allocate roles to nodes automatically as they make sense (i.e. assign
   `mybarclamp-controller` to a one or more designated controller nodes and assign
   `mybarclamp-compute` to all compute nodes).

2. Populate fields that you left blank in your default data bag (e.g. generate
   random passwords).

After adaption our method might look something like this:

```
  def create_proposal
    @logger.debug("Mybarclamp create_proposal: entering")
    base = super

    nodes = NodeObject.all
    server_nodes = nodes.select { |n| n.intended_role == "controller" }
    server_nodes = [nodes.first] if server_nodes.empty?

    compute_nodes = nodes.select { |n| n.intended_role == "compute" }
    compute_nodes = [nodes.first] if compute_nodes.empty?

    base["deployment"][@bc_name]["elements"] = {
      "mybarclamp-controller" => [server_nodes.first.name]
    } unless server_nodes.nil?

    base["deployment"][@bc_name]["elements"] = {
      "mybarclamp-compute" => compute_nodes
    } unless compute_nodes.nil?


    base["attributes"][@bc_name]["database_instance"] =
      find_dep_proposal("database")
    base["attributes"][@bc_name]["rabbitmq_instance"] =
      find_dep_proposal("rabbitmq")
    base["attributes"][@bc_name]["keystone_instance"] =
      find_dep_proposal("keystone")
    base["attributes"][@bc_name]["service_password"] = random_password
    base["attributes"][@bc_name][:db][:password] = random_password

    @logger.debug("Mybarclamp create_proposal: exiting")
    base
  end
```

The Barbican barclamp's `create_proposal` method starts
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L57) 
and ends
[here](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/models/barbican_service.rb#L57).

##### Minimal Controller Class

Location: `crowbar_framework/app/controllers/mybarclamp_controller.rb`

The final thing we will need to the Crowbar application is a controller. This
goes into `crowbar_framework/app/controllers/mybarclamp_controller.rb` and is
very straightforward since we only need stub code in there (we normally don't
do much in Barclamp controllers). This will suffice:

```
class MybarclampController < BarclampController
  # Controller for Mybarclamp barclamp

  protected

  def initialize_service
    @service_object = MybarclampService.new logger
  end
end
```

As you can see, Barbican has the same [piece of boilerplate
code](https://github.com/crowbar/crowbar-openstack/blob/8bebf8a379ebea8ef462ad49746dda6d36a3c46d/crowbar_framework/app/controllers/barbican_controller.rb).

### Local Testing

In the previous sections you created a scaffold for your barclamp, possibly
with some pieces of implementation as well. Before you create a pull request
for that code you should test it locally now. While there is a school of
thought that advocates creating pull requests as soon as the first commit is
in, this approach is problematic for Crowbar development: one of our CI tests
is the [Hound](https://houndci.com/) code style checker, which will generate a
notification for every single violation it detects. Since each commit is likely
to contain multiple such violations, it is better to get your code working
first and then present it to Hound at that point. If you test and debug your
barclamp iteratively while the pull request is already open you will likely
generate a lot of needless spam (e.g. for code that may end up eventually
getting removed entirely in the course of development)

For testing your code locally you will need the development environment
described in the [Prerequisites](#prerequisites) section. So build such an
environment now (if you haven't already done so) and make sure you've got an
up-to-date checkout of your `crowbar-openstack` fork in
`/root/openstack-crowbar` on the development environment's Crowbar admin node.
This is also a good time to create a LVM snapshot or other backup of your
Crowbar admin node in case you break it in the course of debugging.

With your Crowbar node backed up and a checkout of your `crowbar-openstack`
fork in place, apply your code to the running Crowbar instance using the
commands from the [Preparations for Crowbar Node](#preparations-for-crowbar-node)
section:

```
Fpatch   # Fix any merge conflicts reported by this command before proceeding
sync_crowbar
```

If you navigate to your Crowbar Admin Node's web interface now and select
*OpenStack* from the barclamp menu, you should see your new barclamp listed. If
it doesn't appear or you got an error message from the `sync_crowbar` command
you will now have to start debugging. Common errors at this stage are mostly
bad file names, or problems in `mybarclamp.yml`.

#### Create Proposal

Once your barclamp appears in the list of barclamps, the next testing step is
creating a *proposal*: this means creating a run-time configuration for the
Barclamp and saving it.

First of all, click on the `Create` button for your Barclamp in the list of
barclamps. If this fails the reason is usually due to one of the following:

* Errors in `crowbar_framework/app/models/mybarclamp_service.rb`
* Error in `crowbar_framework/app/controllers/mybarclamp_controller.rb`
* Errors in `crowbar_framework/app/views/barclamp/mybarclamp/_edit_attributes.html.haml`
* Errors in your default data bag, e.g.`chef/data_bags/crowbar/template-mybarclamp.json`
  schema`. This is what error messages about invalid JSON syntax usually refer
  to.

Upon failure the Crowbar web UI will usually (but not in all cases) display a
stack trace that will lead you to the problematic file and at least hint at
what the problem is. Once creating the barclamp succeeds you can advance to the
next step.

#### Save Proposal

#### Implement and Debug Chef Recipes

### Pull Request Testing

#### Fix Hound Violations

#### Mkcloud Gating

#### Reviewer approval
