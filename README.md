# _Ansible Essentials_ Course
Ny notes and stuff used while following the [Ansible Essentials](https://www.udemy.com/course/ansible-essentials/) course on Udemy.

Github repo that contains stuff from author is [here](https://github.com/uguroktay/ansible_essentials).

## Section 1: Introduction

Should I use docker? Is there a docker-compose setup available? Vagrant is too heavy...But creating the docker env is extra work and means adding extra challenges. OK, let's do it...[here](./docker/readme.md) is the docker setup & it's explanation.

This is the lab topology my docker network needs to provide:
![Lab Topoology](lab-topology.png)

## Section 2: Ansible Foundations & Installation

### Commands

* `ansible --version` also shows the settings, i.e. where the config file is located etc.
* `ansible-config --view` shows the config that is valid now.
* `ansible-config dump --only-changed`
* `ansible-config --version` also shows which cfg file is used.

The `ansible.cfg` file is in `/etc/ansible/ansible.cfg`. It points to our hosts file in `/home/ansible/hosts`. 

**Note** that both directories `/etc/ansible` and `/home/ansible` of the controller docker are mounted to local directories under `controller-mountpoints`.

**Order of config file locations** is 

* `ANSIBLE_CONFIG` env variable
* `ansible.cfg` in the current dir. This is what we are using, so we need to make sure we alsways start ansible commends from whithin `/home/ansible`
* `~/.ansible.cfg` a hidden file in the home dir
* `/etc/ansible/ansible.cfg` 

## Section 3: Ansible Ad-Hoc Commands

Ansible modules are listed [here](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)(by category)

### Problem: SSH Connection

The controller docker cannot ssh into the other dockers. And this is needed, even for the Ansible ping module (as the ping module creates a SSH connection - instead of actually pinging...).

So I replaced the blank CentOS docker images with a [docker image that contains the SSH service](https://hub.docker.com/r/jdeathe/centos-ssh) - but the SSH service needs additional installing of SSH keys...