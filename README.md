
lovelace-ansible-playbook
=========================

Fork of [django-ansible].

Ansible Playbook for installing the Lovelace learning environment for various purposes.
It installs and configures the following applications that are required to run the Lovelace stack.

- Nginx
- Gunicorn
- PostgreSQL
- Supervisor
- Virtualenv
- Redis
- Celery
- RabbitMQ

Depending on configuration these will either be installed in the same machine, or into separate
machines, with redundancy where necessary. In addition to installing the requirements, it will also
install Lovelace itself, and has an option to install PySenpai - a checking module used in
combination with Lovelace to check Python and C exercises.

Current installation options are:
- Vagrant for local testing and demo

## Installing Lovelace

### Demo Installation with Vagrant

Requirements:
- [Ansible][ansible-installation_guide]
- [Vagrant][vagrant-downloads]
- [VirtualBox][virtual-box_downloads]

Tested on Ubuntu 22.04

This option will install Lovelace into a virtual machine managed by Vagrant. Configuration is
stored in `group_vars/vagrant/vars.yml`. If you don't have any special requirements you can simply
use the provided settings and boot up a demo installation from the root folder of this repo with

```
vagrant up
```

When running post-installation actions, you can give tags like this:

```
ANSIBLE_ARGS="--tags=optionaltag" vagrant provision
```



### Development Installation

TBD

### Production Installation into Multiple Servers

TBD


## Post-Installation Actions

The playbook has several actions that are not inteded to be run every time but rather only once, or
optionally.

### One-Time Initial Actions

Some preparations should only be run once after installation. In order to do so, run the playbook
with `lovelace.initial` tag. Currently this does the following:

- creates a superuser with credentials provided in the configuration

### Installing Checking Environment

The checking environment is not installed by default because it's not exactly a component of
Lovelace. In order to install it you can run provision with the `pysenpai` tag. This creates
a virtualenv for checking, and installs PySenpai into the virtualenv with all its optional
components. This should only be ran for servers that will act as task workers.

### Importing Content

If you want to import content into a fresh or existing Lovelace instance, you can configure
the `content_import_list` parameter in `group_vars` to include a list of exported zip files
that will be imported into the instance. Importing content this way is idempotent. Note that
this import is done with the Django manage command that uses superuser privileges for importing
meaning it will overwrite any existing content that has the same natural key.

To import content, run provision with the `lovelace.import` tag after configuring

### Updating Lovelace

When updating Lovelace, instead of running full provision, use the `deploy` tag.

[django-ansible]: https://github.com/jcalazan/ansible-django-stack/
[ansible-installation_guide]: https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
[vagrant-downloads]: https://www.vagrantup.com/downloads.html
[virtual-box_downloads]: https://www.virtualbox.org/wiki/Downloads

