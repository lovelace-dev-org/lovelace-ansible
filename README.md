
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

## Ansible Galaxy Roles

Install the Redis role

```
ansible-galaxy install geerlingguy.redis
```


## Installing Lovelace

### Demo Installation with Vagrant

Requirements:
- [Ansible][ansible-installation_guide]
- [Vagrant][vagrant-downloads]
- [VirtualBox][virtual-box_downloads]

Tested on Ubuntu 22.04

This option will install Lovelace into a virtual machine managed by Vagrant. Configuration is
stored in `group_vars/development/vars.yml`. If you don't have any special requirements you can simply
use the provided settings and boot up a demo installation from the root folder of this repo with

```
vagrant up
```

When running post-installation actions, you can give tags like this:

```
ANSIBLE_ARGS="--tags=optionaltag" vagrant provision
```


### Development Installation for Localhost

Requirements:
- Ubuntu (or compatible OS that uses apt for installations)
- [Ansible][ansible-installation_guide]

This installation method installs all the requirements and runs them, but does not run Lovelace itself
or the Celery workers. Running them manually gives more direct access to debug information. If you want to run
them automatically with Supervisor, you can remove the `run_manually` variable from `local.yml` (or set it to false).

Be aware that this creates a lot of changes on your local computer. Therefore creating a separate virtual machine for development is recommended. The user executing the command must be in sudoers.

```
ansible-playbook -i hosts.yml local.yml
```

This installation creates two virtual envs `/opt/lovelace/` and `/checkers/python/`, and clones the Lovelace
repository into `/opt/lovelace/lovelace/`. Ownership of the repository is transferred from the lovelace user
to the user running the playbook to prevent ownership issues when using git.

In order to be able to run Lovelace from the command line manually, you need to activate the virtualenv, and export the environment variables that are needed for configuration.

```
source /opt/lovelace/bin/activate
source /opt/lovelace/bin/postactivate
```

After this you can use Django's manage

```
cd /opt/lovelace/lovelace/webapp
python manage.py runserver
```

and manually start workers with Celery (need to run as root for demotion to work)

```
cd /opt/lovelace/lovelace/webapp
celery -A lovelace worker -Q default --loglevel=info -n checker1@%h
```


### Production Installation into Multiple Servers

TBD


## Post-Installation Actions

The playbook has several actions that are not inteded to be run every time but rather only once, or
optionally.

### One-Time Initial Actions

Some preparations should only be run once after installation. In order to do so, run the playbook
with `lovelace.initial` tag. Currently this does the following:

- creates a superuser with credentials provided in the configuration

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

