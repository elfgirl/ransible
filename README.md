# Raspberry Pi Ansible Playbook

Adrianne Mulvaney

Heavily based off of https://github.com/glennklockwood/rpi-ansible with the following tweaks and fixes 

* More focused on local/pull models so nodes can self configure. 
* Listing of machine specific git repositories to clone for assigned projects
* Bootstrap section to enable setup of ansible-pull on a cron schedule plus boot
* Setup of supervisor and uwsgi for projects to execute and persist
* Python setup and general housekeeping
* Some better logging

# Introduction

This is an Ansible configuration that configures a fresh Raspbian installation
on Raspberry Pi.  It can be run in local (pull) mode, where ansible is running
on the same Raspberry Pi to be configured, or standard remote mode. 

# Using Bootstrap Mode (configure pull and self-configuring options)

Flash your microSD with the latest Rasbain flavor of choice (Lite is recommended)
Configure your headless network via these docs https://desertbot.io/blog/headless-raspberry-pi-4-ssh-wifi-setup You will only need to follow the SSH and WiFi steps. Keygen, bonjour and the rest are handled in the bootstrap.yml

Update `repo_url` in bootstrap.yml to point to your git repository. Rememeber the ansible-pull playbook must be named `local.yml`. Setting up a PAB for private repositories is left as an exercise for the readers

* Boot your Pi and log in via the default credentials `pi@raspberrypi.local` and `rasberry`. 
* Run `ifconfig` and record the MAC address (or read it from route tables on your router)
* Update the host_vars/hostmaps.yml with the MAC address and hostname
* Create a `hostname.yml` file in `host_vars` with the desired options (see ##Configuration for specifics)
* To add local users, create and edit `roles/rpi/vars/users.yml`.  Follow the structure in `roles/rpi/vars/users.yml.example`. 
* Commit the updates

*NOTE* this will reboot your Pi once to start the auto-config pull

```shell
    (ansible_env) $ ansible-playbook ./bootstrap.yml -t all --skip-tags "python"
```  

## Bootstrap Tags

Several tags are configured for customizing the bootstrap process with includes/excludes

* `all` Configure with all the below tags
* `pull` Sets up the cron jobs to pull the repository and set self-configuration execution schedule. 
* `sshd` Generates new sshd keys
* `sw` Installs basic software like ntp, avahi-daemon, git. Nothing Raspery Pi specific as that is in the rpi role
* `users` Sets up any configured users and their ssh keys. These are not regular users but users like `ransible` for internal use
* `python` Installs an alternative python via pyenv. Note pyenv compilation can take a long time on Zero and low powered machines. The default alternative is currently 3.9.5 however any version can be specified via command line argument extra arguments `--extra-vars "pyenv_python_version=3.9.10`


# Using Remote Mode (standard push model)

Update the hosts file with the desired inventory
Run one of the playbooks as you would do normally:

```shell
    (ansible_env) $ ansible-playbook ./rpi.yml -t all 
```

The default hosts file and `become_*` configurations are set in ansible.cfg.

# Using Local Mode (manual pull and updating) instead of Bootstrap

Edit `host_vars/host_maps.yml` and add the mac address of `eth0`/`wlan0` for the Raspberry Pi to the `macaddrs` variable.  Its key should be a mac address (all lower case) and the value should be the short hostname of that system.  Each such entry's short hostname must match a file in the `host_vars/` directory.

Then run the playbook from the Rasberry Pi :

```
    (pi_ansible_env) $ ansible-playbook ./local.yml
```

The playbook will self-discover its settings, then idempotently configure the Raspberry Pi.


# Host Configuration Options

The contents of each file in `host_vars/` is the intended configuration state
for each Raspberry Pi.  Look at one of the examples included to get a feel for
the configurations available.  Most variables should be optional, and if left
undefined, do not enforce a specific configuration.

To add local users, create and edit `roles/rpi/vars/users.yml`.  Follow the
structure in `roles/rpi/vars/users.yml.example`.  You can/should
`ansible-vault` this file.


## After running the playbook

This playbook purposely requires a few manual steps _after_ running the playbook
to ensure that it does not lock you out of your Raspberry Pi.

1. `usermod --lock pi` to ensure that the default user is completely disabled.


## SSH host keys

This playbook can install ssh host keys.  To do so,

1. Drop the appropriate `ssh_host_*_key` files into `roles/common/files/etc/ssh/`
2. Rename each file from `ssh_host_*_key` to `ssh_host_*_key.{{ ansible_hostname }}`
3. `ansible-vault encrypt roles/common/files/etc/ssh/ssh_host_*_key.*`

The playbook will detect the presence of these files and install them.

# Ansible Role Overviews

## Bootstrap

This file sets up some initial files, crons jobs, ntp and other such basic linux materials. It is run on a freshly flashed image to prepare it. Project or role specific material is not set up here

## Common 

This sets up common operations to all Raspberry Pis such a gpio-zero. Lightweight process   

## Rpi

This is the big workhorse. Conceptually a lot of items could go into Common such as hostname, time_zone etc. However this is where they live now

This is also where a lot of the host specific material such has host_apt_software, host_pip_software, and host_ports are implimneted. 

Two components of this playbook deserve further explanation as they provide they ways to get specific software running on specific hosts. 

## Supervisord 

In each host yml file, there is section at the end called `host_supervisor_apps` which is a dictionary list of git repositories to clone and set up for automatic execution with supervisor. This is a way to make sure your pi project is always running

Each repo in the list is 

* Cloned into the specified users home directory
* `pipenv --bare install --dev` is executed on the project to install local dependencies, if this is not a python project using pipenv.... enjoy tweaking it. 
* a supervisor conf file is placed in /etc/supervisor/conf.d and the process is alerted to re-read and update

Example :

```

host_supervisor_apps :
  - name : waterer
    app_exec : waterer_server.py 
    app_user : pi
    repo : 'https://elfgirl:{{pinecrest_pat}}@github.com/elfgirl/pinecrest.git'

```

Pipenv environment for the project is a must as the command `pipenv run` is hardcoded. This ensures each project has its own environment

TODO: Private repos need a PAT. Need to juggle that in vault for the ansible pull local side of things

Warnings are from the privlege de-escalation into user `pi` for execution. 

## Web Server

Sets up gunicorn/nginx for projects that expose out uWSGI applications. Example of `sensor-server.py` for serving out RESTful sensor information

## Acknowledgment

I stole a lot of knowledge from https://github.com/glennklockwood/rpi-ansible
