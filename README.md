
Lovelace Ansible Playbook
=========================

Fork of [django-ansible].

Ansible Playbook for installing the Lovelace learning environment for various purposes.
It installs and configures the following applications that are required to run the Lovelace stack.

- Nginx
- Gunicorn
- PostgresQL
- Supervisor
- Virtualenv
- Redis
- Celery
- RabbitMQ

It can also create its own CA and distribute self-signed certificates on the hosts for enabling
encrypted communication and certificate based authentication. It is also possible to configure a
central logserver, and have all hosts send their logs there.

Depending on configuration these will either be installed in the same machine, or into separate
machines, with redundancy where necessary. In addition to installing the requirements, it will also
install Lovelace itself, and has an option to install PySenpai - a checking module used in
combination with Lovelace to check Python and C exercises.

Current installation options are:
- Vagrant for local testing and demo
- Development installation on localhost
- Production installation on multiple hosts

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
sudo ansible-playbook -i hosts.yml local.yml
```

If you are using a virtualenv for Ansible, you need to pass its environment variables to sudo:

```
sudo -E env PATH=$PATH ansible-playbook -i hosts.yml local.yml
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
sudo -E env PATH=$PATH celery -A lovelace worker -Q default --loglevel=info -n checker1@%h
```


### Stand Alone Main Server Installation

Requirements:
- Ubuntu server

This method will install a complete Lovelace installation into one server. This only installs the main server. If you want to run checkers, they need to be installed separately, or by running a full production installation (separate document).

All the templates mentioned in this section are available in the templates folder of this repository.

This installation will use certbot to get certificates. You will need to modify the process if you want to use your own certificates.

#### Preparing the Server

This guide assumes you are logging into the server with SSH key authentication, and have added the private key to your SSH agent. Your username that you log in with must have sudo access on the server as well.

#### Preparing Inventory

Copy the `inventory_simple.yml` template file into a suitable location. If you didn't clone this repository for developing it, you can put it in the root folder. Otherwise put it into a folder that is outside of the repository. For installing the main server only, you only need to replace `lovelace.placehold.er` with your server's address into the lovelace group. E.g.

```
lovelace:
  hosts:
    example.com:
      gunicorn_num_workers: 2
      gunicorn_max_requests: 0
      gunicorn_timeout_seconds: 300
```

You should also adjust the `gunicorn_num_workers` parameter to a number suitable for your server (suggested default is 2 * (number of cores) + 1).

#### Preparing Vault

The configuration pulls a lot of variables from an additional variables file that shuold be encrypted using Ansible vault. Create a new vault with

```
ansible-vault create /path/to/vault
```

If you are not doing development, you can place the vault to `group_vars/simpleprod/vault.yml` and it should be automatically included. Otherwise place it outside your repository, and give its location to explicitly when running the playbook.

Then paste the contents from the vault.yml template, and fill in the following fields:

- `vault_db_password` - this will be your database user's password
- `vault_django_secret_key` - django secret key that will be used to generate session keys etc.
- `vault_django_superuser_name` - username for the initial django superuser
- `vault_django_superuser_pass` - superuser password
- `vault_django_superuser_email` - superuser email address
- `vault_django_admins` - comma-separated string with at least one admin as "firstname lastname email"
- `vault_smtp_host` - SMTP host address

If you are going to use your own certificates, paste the certificate under `vault_nginx_ssl` and its private key under `vault_nginx_key`. To use them, you need to override the `nginx_use_letsencrypt` variable to false when running the playbook.

#### Running the Playbook

We are going to run the `simpleprod.yml` playbook. The example assumes both files were placed outside the repository.

```
ansible-playbook -i /path/to/inventory.yml -e @/path/to/vault -e server_user=username -J -K simpleprod.yml
```

In this example we are overriding the server_user variable with the username. You can override other variables as needed in the same way. This command will prompt for your vault password and your sudo password. If you also need to prompt for the SSH password, add the `-k` flag. For more complicated use cases, please refer to the [ansible-playbook script documentation][ansible-playbook_docs].

After running the script, you should be able to see the Lovelace front page when pointing your browser at the server's address.


### Websocket Backend Server Installation

Requirements:
- Ubuntu server

This method will install the Lovelace server in websocket mode. These backend servers are used by interactive widgets that need to run student code and show the results. The process is largely similar to the standalone web server installation above.

#### Preparing Inventory

Copy the `inventory_simple.yml` template file into a suitable location. If you didn't clone this repository for developing it, you can put it in the root folder. Otherwise put it into a folder that is outside of the repository. For installing the websocket server, you only need to replace `websockets.placehold.er` with your server's address into the lovelace group. E.g.

```
websockets:
  hosts:
    example.com:
      daphne_processes: 1
```

You can adjust `daphne_processes` but we've found quite solid success with just 1 in the past.

#### Preparing Vault

The configuration pulls a lot of variables from an additional variables file that shuold be encrypted using Ansible vault. Create a new vault with

```
ansible-vault create /path/to/vault
```

If you are not doing development, you can place the vault to `group_vars/simpleprod/vault.yml` and it should be automatically included. Otherwise place it outside your repository, and give its location to explicitly when running the playbook.

Then paste the contents from the vault.yml template, and fill in the following fields:

- `vault_django_secret_key` - django secret key that will be used to generate session keys etc.
- `vault_ws_ticket_host` - address of the Redis server that stores websocket authencication tickets
- `vault_client_crt` - certificate that's been signed with the same CA as the Redis server, for peer cert authentication
- `vault_client_key` - key of the above cert
- `vault_ca_crt` - CA certificate for peer authentication

If you are going to use your own server certificates for NGINX, paste the certificate under `vault_nginx_ssl` and its private key under `vault_nginx_key`. To use them, you need to override the `nginx_use_letsencrypt` variable to false when running the playbook. Note these are different from the certificate used for peer authentication.


#### Running the Playbook

We are going to run the `ws_servers.yml` playbook. The example assumes both files were placed outside the repository.

```
ansible-playbook -i /path/to/inventory.yml -e @/path/to/vault -e server_user=username -e lovelace_main_host=lovelace.addre.ss -J -K ws_servers.yml
```

In this example we are overriding the server_user variable with the username. You can override other variables as needed in the same way. This command will prompt for your vault password and your sudo password. If you also need to prompt for the SSH password, add the `-k` flag. For more complicated use cases, please refer to the [ansible-playbook script documentation][ansible-playbook_docs].

We are also setting the `lovelace_main_host` variable to point to the Lovelace server that you want to allow connections initiated by. You will not need this variable if you are installing both servers from the same inventory file, as its default value will be taken from the

After running the script, your websocket






### Production Installation into Multiple Servers

See the prodocution installation document for details.


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
[ansible-playbook_docs]: https://docs.ansible.com/projects/ansible/devel/cli/ansible-playbook.html
[vagrant-downloads]: https://www.vagrantup.com/downloads.html
[virtual-box_downloads]: https://www.virtualbox.org/wiki/Downloads

