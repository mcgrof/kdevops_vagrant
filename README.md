kdevops_vagrant
===============

`kdevops_vagrant` is an ansible role which deploys a community shared kdevops
Vagrant file into your project. This ansible role copies the Vagrantfile
project namespace directory, and can optionally run vagrant for you.

The goal behind this ansible role is to allow the ability to *share* the same
scalable Vagrantfile easily between projects, and gives the Vagrantfile a home
to allow contributors to advance it, and document it.

Project namespace directory
---------------------------

The project namespace directory is the top level directory of your project,
which defines your project. For instance, say you have a development project
called kdevops, and you have all files for it stored under:

```
/home/user/devel/kdevops/
```

In this case the project namespace directory is `kdevops`. By default this
ansible role infers which is your project namespace directory by using the
basename of the directory where your playbook file is stored which called
this role. For instance, say your playbook file is stored under:

```
/home/user/devel/kdevops/playbooks/kdevops_vagrant_nodes.yml
```

You can override data in another file:

```
/home/user/devel/kdevops/playbooks/kdevops_vagrant_nodes_override.yaml
```


The directory where your playbook file is stored is:

```
/home/user/devel/kdevops/playbooks/
```

The basename of that directory is:

```
/home/user/devel/kdevops/
```

This would be the inferred project namespace directory for this project.
You can override this by using the `force_project_dir` role variable
described below.

Project namespace
-----------------

The project namespace is used within the Vagrantfile carried by this role
to allow you to override defaults used for vagrant through environment
variables. The project namespace is inferred from the project namespace
directory described above. For instance, in the above case where the
project directory namespace is:

```
/home/user/devel/kdevops/
```

The project namespace is inferred to be `kdevops`. The upper case version of
this is used a prefix for environment variables, so `KDEVOPS`. Dashes are
substituted with underscores as shell variables cannot contain dashes. So if
the project directory namespace was:

```
/home/user/devel/some-cool-project/
```

The project namespace would be: `SOME_COOL_PROJECT`

Requirements
------------

You must have vagrant installed.

Vagrant is used to easily deploy non-cloud virtual machines. Below are
the list of providers supported:

  * Virtualbox
  * libvirt (KVM)

The following Operating Systems are supported:

  * OS X
  * Linux

## Running libvirt as a regular user

We makes use of the [https://github.com/mcgrof/libvirt-user](libvirt-user)
ansible role to enable a regular user. Read the documentation there for
the logic into that.

Overriding node configuration with a different file
----------------------------------------------------

The better way to support overriding variables used in vagrant is throught
the `$(project)_node_override.yaml` file. So for example, for the
[https://github.com/mcgrof/kdevops](kdevops project), you can override data
in another optional file kept outside of git:

```
/home/user/devel/kdevops/playbooks/kdevops_vagrant_nodes_override.yaml
```

Environment variables
---------------------

Only if using a override file does not work you can use environment variables.
But using environment variables is done on a case by case basis, given that
we have to implement support for each new variable added. Using the overrides
file, you can override anything, as soon as a new feature is added.

Environment variables are prefixed by the upper case version of the project
namespace as described above. Let us refer to this as `${PN_U}`. Taking this
prefix into consideration, the following environment variables modify the way
the Vagrantfile works:

  * `${PN_U}_VAGRANT_NODE_CONFIG`: lets you override default configuration
    file used
  * `${PN_U}_VAGRANT_PROVIDER`: lets you override provider used by vagrant
  * `${PN_U}_VAGRANT_LIMIT_BOXES`: lets you enable box limitting
  * `${PN_U}_VAGRANT_LIMIT_NUM_BOXES`: number of boxes to limit
  * `${PN_U}_VAGRANT_QEMU_GROUP`: overrides the qemu user to use

## Limitting vagrant's number of boxes

By default vagrant will try to create *all* the nodes specified on your node
configuration file. By default this is `${PN_L}_nodes.yml` where `$PN_L` is the
lower case version of your project namespace described above. For example, for
the kdevops projec this would be kdevops_nodes.yml.

For example if you want to limit vagrant to only create *one* box on the
kdevops project you would use:

```bash
export KDEVOPS_VAGRANT_LIMIT_BOXES="yes"
export KDEVOPS_VAGRANT_LIMIT_NUM_BOXES=1
```

You'd obviously would use a different prefix for differently named projects.

This will ensure only the first host, for example, would be created and
provisioned. This might be useful if you are developing on a laptop, for
example, and you want to limit the amount of resources used.

Scope of use of ansible
-----------------------

`kdevops` as a whole is designed to be agnostic to how you set up a system,
whether that be a virtualized local systemw it libvirt / Virtualbox, you use
a bare metal system, or you use a cloud environment.

Because of this the there 3 stages to bring up:

  * Bring up: virtual / cloud / bare metal
  * Configure and install dependencies: add systems to ~/.ssh/config, and install
    users's preferred bash scripts, .gitconfig, and generic development packages
    of choice. This is handled through the
    [http://github.com/mcgrof/update_ssh_config_vagrant](update_ssh_config_vagrant)
    and
    [http://github.com/mcgrof/devconfig](devconfig)
    ansible roles.
  * Let it rip: do your work, and this is typically now encouraged through
    ansible

Our use of vagrant with ansible is limited to the first two parts mentioned
above. And so we use ansible *only* to use the two ansible roles:

  * [http://github.com/mcgrof/update_ssh_config_vagrant](update_ssh_config_vagrant)
  * [http://github.com/mcgrof/devconfig](devconfig)

We purposely don't want to extend this practice, and the reason is that we
want users to use ansible directly for later objectives. The work behind
bring up and configuring, installing developer dependencies must be handled
either by vagrant, terraform, or the user who set up the bare metal systems
manually.

Because of this we don't encourage adding more ansible roles for vagrant's use.
These two roles should suffice for typical setups, and the *rest* of the use
of ansible should be done manually.

Ansible python interpreter
--------------------------

Ansible relies on python on remote systems. Which version of python should
be used will vary, but by default ansible will try to use an old version of
the python interpreter, so to help support older systems. This decision
isn't the best for newer systems, and so you are encouraged to specify
the interpreter. This best set on the inventory file. The Vagrantfile
assumes you have an inventory file called ../hosts, you however can override
this on the ansible configuration as follows:

```
ansible_playbooks:
  # If this file exists you can override any ansible variable there.
  # This file is optional.
  extra_vars: "../extra_vars.yml"
  inventory: "../hosts"
  playbooks:
    - name: "../playbooks/update_ssh_config_vagrant.yml"
    - name: "../playbooks/devconfig.yml
```

Your hosts file might look like something like this:

```
[all]
newsystem1
oldsystem1
[all:vars]
ansible_python_interpreter =  "/usr/bin/python3"

[new]
newsystem1
[new:vars]
ansible_python_interpreter =  "/usr/bin/python3"

[old]
oldsystem1
[dev:vars]
ansible_python_interpreter =  "/usr/bin/python2"
```

In this case all does set `[all]` group the python interpreter to python3,
however the last entry will ensure that the old system uses python2 instead.

Ansible extra vars use with vagrant
-----------------------------------

Ansible has its own heirarchy of how variables take precedence, through
either the command line, or role files, etc, and this is documented on
the [https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html](ansible playbooks variable documentation page).

While this might suit most uses, it doesn't really help with projects which
want to allow users specify their own variations to the defaults in a file
not present in version control, and without having to expand on the command
line.

Since kdevops consists in a unified set of ansible roles we support a unified
way to define overrides to ansible. We do this by allowing the user to override
ansible variables in the project root directory in a file called:

  * `extra_vars.yml`

So in the kdevops project example, this would be:

```
/home/user/devel/kdevops/extra_vars.yml
```

When this file is set the vagrant ansible plugin will ensure the just the file
is passed on to ansible directly on the command line via `--extra-vars=@file`.
The prefix of `@` is required when specifying a file. You don't have to provide
the `@`, we do that for you.

All of the ansible roles used with kdevops supports looking for this
`extra_vars` file.

Ansible role Variables
----------------------

  * run_vagrant: by default is False, if set to True we will run Vagrant for you
  * force_project_dir: if set, this will be used as the directory where we will
    look for the vagrant directory in, and eventually copy the Vagrantfile. By
    default this is set to empty, and we infer your project directory to be
    the parent directory of where playbook file resided.

Dependencies
------------

None.

Example Playbook
----------------

Below is an example playbook, it is used on the kdevops project,
so kdevops/playbooks/kdevops_vagrant.yml file:

```
---
- hosts: localhost
  roles:
    - role: kdevops_vagrant

```

In this particular case note how localhost is used. This is because we are
provisioning the Vagrantfile to the kdevops/vagrant/ directory locally.
You could obviously use a different host.

Further information
--------------------

For further examples refer to one of this role's users, the
[https://github.com/mcgrof/kdevops](kdevops) project or the
[https://github.com/mcgrof/oscheck](oscheck) project from where
this code originally came from.

License
-------

GPLv2
