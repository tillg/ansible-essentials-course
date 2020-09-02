# _Ansible Essentials_ Course

Ny notes and stuff used while following the [Ansible Essentials](https://www.udemy.com/course/ansible-essentials/) course on Udemy. My setup differs a bit from the one suggested in the course as I use a docker network rather than a Vagrant one.

Github repo that contains stuff from author is [here](https://github.com/uguroktay/ansible_essentials).

## Section 1: Introduction

Should I use docker? Is there a docker-compose setup available? Vagrant is too heavy...But creating the docker env is extra work and means adding extra challenges. OK, let's do it...[here](./docker/readme.md) is the docker setup & it's explanation.

This is the lab topology my docker network needs to provide:
![Lab Topoology](lab-topology.png)

## Section 2: Ansible Foundations & Installation

### Commands

- `ansible --version` also shows the settings, i.e. where the config file is located etc.
- `ansible-config --view` shows the config that is valid now.
- `ansible-config dump --only-changed`
- `ansible-config --version` also shows which cfg file is used.

The `ansible.cfg` file is in `/etc/ansible/ansible.cfg`. It points to our hosts file in `/home/ansible/hosts`.

**Note** that both directories `/etc/ansible` and `/home/ansible` of the controller docker are mounted to local directories under `controller-mountpoints`.

**Order of config file locations** is

- `ANSIBLE_CONFIG` env variable
- `ansible.cfg` in the current dir. This is what we are using, so we need to make sure we always start ansible commends from whithin `/home/ansible`
- `~/.ansible.cfg` a hidden file in the home dir
- `/etc/ansible/ansible.cfg`

## Section 3: Ansible Ad-Hoc Commands

General syntax for invoking ad hoc commands:

```bash
ansible -i <inventory_file> <target_hosts> -m <module_name> -a 'arguments'
```

Ansible modules are listed [here](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)(by category). Examples are

- Installing things using `yum` or `apt`
- Starting or stopping services
- Copying files
- Doing things on AWS
- ...

To start the docker compose network and run some ad hoc commands:

```bash
cd docker
docker-compose up
docker exec -it controller bash

ansible all -m ping

ansible web2 -m command -a "uptime"
# same as this (because if no module is specified "command" is assumed):
ansible web2  -a "uptime"

# Get the name of our 2 web servers:
ansible web -a "uname -a"

# Copy the hosts file to tmp an both web servers:
ansible web -m copy -a "src=/etc/hosts dest=/temp/"

# Do the same thing for servers in our lb group and our web group (":" means "or"):
ansible lb:web -m copy -a "src=/etc/hosts dest=/temp/"

# Install the ntp package on both our web servers:
ansible web -m apt -a "name=ntp state=latest"
# ...and delete it again:
ansible web -m apt -a "name=ntp state=absent"

# See what Ubuntu versions your nodes are running (Note that we use shell instead of command):
ansible all -m shell -a "cat /etc/*release"

# Get lots of informatoion about your web1 server:
ansible web1 -m setup

# How much memory do my machines have?:
ansible all  -m setup -a "filter=ansible_memtotal_mb"
```

### Problem: SSH Connection

The controller docker at first could not ssh into the other dockers. And this is needed, even for the Ansible ping module (as the ping module creates a SSH connection - instead of actually pinging...).

**Solution**
I replaced the CentOS docker images with a Ubuntu image, as I know more about Ubuntu. Then I created Dockerfiles for the controller and for the nodes that make sure that the SSH server and all the needed packages are installed. I also created start scripts that start the SSH server with the parameters needed (i.e. root user can login via ssh).

**Careful**, I changed the password of root users to `root` to make things easier. But it's not safe! 🤞

Helpful commands:

- See what services are running: `service --status-all`
- Check if ssh deamon is running: `service ssh status`
- Start ssh deamon: `service ssh start`
- What Linux version am I using?: `lsb_release -a`

## Section 4: Ansible Playbooks

### YAML

- Good practice: Use 2 spaces for indentation, don't use tabs.
- I use the YAML VScode extension (YAML Language Support by Red Hat)

### Tasks

- Tasks are tasks that are performed by Ansible 😂
- We describe tasks by _the state that should be achieved after execution_.
- In the course they mention a lot the `yum` module to install packages. Since I don't use CentOS but Ubuntu, I need to use `apt` instead. But it seems smart to use the more abstract `package` module that is available since Ansible 2. For an example see (here)[https://serverfault.com/questions/587727/how-to-unify-package-installation-tasks-in-ansible].

### Playbooks

- Playbooks are made out of plays - duuuh...

To start the docker compose network and run some ad hoc commands:

```bash
cd docker
docker-compose up
docker exec -it controller bash

# Go to the dir where we have the playbook
cd /home/ansible/

# Check the syntax of the playbook
ansible-playbook playbook1.yml --syntax-check

# Do a dry run
ansible-playbook playbook1.yml --check

# If you want to read more details
ansible-playbook playbook1.yml --check -v#

#...even mnore details:
 ansible-playbook playbook1.yml -C -vv

 # Run it for real:
 ansible-playbook playbook1.yml

# Run it and see all the SSH communication:
ansible-playbook playbook1.yml -vvv
```

### Setup module / Host facts

There is a `setup` module that gathers facts about hosts. That's what ansible runs at tghe beginning of the playbooks.

Some examples:

```bash
ansible web1 -m setup -a "filter=ansible_os_family"

#Result would be (in our case, running on Ubuntu)
web1 | SUCCESS => {
    "ansible_facts": {
        "ansible_os_family": "Debian"
    },
    "changed": false
}
```

Some intersting variables provided by the setup module:

![Setup module variable](ansible-facts.png)

The setup module is pre-pended to every playbook by default. It's reported as _Gathering facts_. As gathering facts might take quite some time, you can disable it for a playbook like so:

```yaml
- hosts: all
  gather_facts: false
  tasks:
     - name: ...
```

### Package module 

In the course they use the `yum` module, but since we are on a Debian based environment...

The docume ntation of the package module [is here](https://docs.ansible.com/ansible/latest/modules/package_module.html). Note: It seems way easier than the yum module 😂

Run the stuff:

```bash
cd docker
docker-compose up &
docker exec -it controller bash

cd /home/ansible

# Check our playbook
ansible-playbook web.yml --syntax-check

# Run it
ansible-playbook web.yml 
```

### Service Module

The service module is used to start, stop services.

```bash
# Get help about ansible modules on the command line:
# ansible-doc <module>
ansible-doc service
```

Check if the web server runs by pointing your browser to it: On your host machine, got to [http://localhost:1011/](http://localhost:1011/)

### Loops

A sample loop:

```yaml
- name: Copy using inline data
    copy:
    src: "{{ item }}"
    dest: /var/www/html/
    owner: root
    group: root
    mode: 0644
    backup: yes
    loop:
    - files/index.html
    - files/index.php
```

Alternative use `with_<lookup>` like so:

![With Items loop](ansible_with_items_sample.png)

Possible `with_<something>` loops are:

![With-something loops](ansible_with_lookup.png)

The recommended way to install multiple packages (and other ways are often suggested when searching Google):

![How to install multiple Packages](ansible_multi-paket_recommendation.png)
 
 **Note:** I still used the `loop` construction, since I used the `package` module (that doesn't allow to list multiple packages)

 Check if copying the php file, ionstalling & restarting all the services worked by checking [the web server on web2](http://localhost:1012/index.php)

 ### Variables

* Variables alwas start with a letter
* Variables conatin letters, numbers, underscores
* What doesn't work are dashes, dots...

The `debug` module is great for checking variables:

```YAML
- name: Output variable hello
  debug:
    msg: "{{ hello }}"
```

Variables can also be **lists**:

```YAML
vars:
  countries: [Korea, China, Russia, India]

# or:

vars:
  countries:
  - Korea
  - China
  - Russia
  - India
```
Variables can also be **dictionaries**:

```YAML
- hosts: web1
  vars:
    USA: { Capital: 'Washington DC', Continent: 'NA',President: 'Donald Trump' }
  tasks:
  - name: Ansible Dictionary output items
    debug:
      msg: "{{ USA.President }} is equivalent to  {{ USA['President'] }}"
    loop: "{{ USA|dict2items }}"```
  - name: Ansible Dictionary dict2Item
    debug:
      msg: "Key is {{ item.key }} value is {{ item.value}}"
    loop: "{{ USA|dict2items }}"
```

This is what is actually happening here:

![dict2items](ansible-dict2items.png)

There are many different places where you can declare variables (sorted by precedence):

![Ways to declare variables](ansible_where_to_decrae_vars.png)

* In a playbook in a `vars:` section
* In a playbook as external variables file:

```YAML
- hosts: web1
  vars_files:
  - /vars/eternal_vars.yml
  tasks:
  ...

# vars/external_vars.yml
somewhar: somevalue
password: magic
```

* In a file per host:

```YAML
- hosts: all
  include_vars: "{{ ansible_hostname }}.yml"
  tasks:
```

* Thru a user prompt:

```YAML
- hosts: web
  gather_facts: false
  vars_prompt:
  - name: "version"
    prompt: "Which version do you want to install?"
  tasks:
  - name: Ansible prompt example.
    debug:
      msg: "will install httpd-{{ version }}"
  - name: install specific apache version
    yum: 
      name: "httpd-{{ version }}"
      state: present 
```

* Inventory variables:
  * Defined in the inventory file
  * Files located under `group_vars` and `host_vars` folder. 
  * Each file should be named after the host or group it belongs to.
  * Define site wide defaults inside `group_vars/all` file.
  * If the `group_vars` and `host_vars`folders are located in the same folder as the inventory (`hsosts`file), they apply also to ad hoc commands. Place the in the same directory as playbooks to make them available to playbooks only.

```YAML
# File group_vars/web
vhost_config_file: /etc/httpd/conf.d/ansible.conf
apache_config_file: /etc/httpd/conf/httpd.conf
document_root_path: /var/www/ansible/
repository: https://github.com/uguroktay/ansible_essentials.git
check_interval: 1
```

```YAML
# file: ansible/group_vars/all
# This is the site wide default
ntp_server: default-time.example.com
apache_port: 80
```
```YAML
# file: ansible/group_vars/europe
# This is the default for the group named europe
ntp_server: europe-time.example.com
apache_port: 8080
```
```YAML
# file: ansible/host_vars/web1
# This is the default for host web1
ntp_server: bucharest.example.com
apache_port: 80
```

* Set variables on the command line

```bash
# Launch a ansible playbook like so:
ansible-playbook extravars.yml --extra-vars "my_message='extra vars are powerful' other_var='something else'"
```

### Variables precendence

**Rule**: Avoid defining the same variable at different places!

Ansible has 3 main scopes:
* Global: This is set by config, environment variables and the command line
* Play
* Host

### Lineinfile module

**Target**: Have a playbook that updates /etc/hosts file

```YAML
---
- name: common tasks
  hosts: all
  become: true
  tasks:
    - name: update/etc/hosts file
      lineinfile:
        path: "{{ hostsfile }}"
        regexp: "{{ item }}$"
        line: "{{ hostvars[item].ansible_facts.eth1.ipv4.address }} {{ hostvars[item].ansible_facts.hostname }}.{{ domainname }} {{ hostvars[item].ansible_facts.hostname }}"
      loop: "{{ groups['all'] }}"
```
```YAML
# file: group_vars/all
domainname: "testdomain.com"
hostsfile: "/etc/hosts"
```

