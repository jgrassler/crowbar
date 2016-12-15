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

### Basic implementation

#### Data Bag

#### Roles

#### Chef Cookbook

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
