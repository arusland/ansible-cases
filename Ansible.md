# Ansible

Ansible is an open-source software provisioning, configuration management, and application-deployment tool. It runs on many Unix-like systems, and can configure both Unix-like systems as well as Microsoft Windows. It includes its own declarative language to describe system configuration.

It was written by Michael DeHaan and acquired by Red Hat in 2015. Unlike competing products, Ansible is agentless - temporarily connecting remotely via SSH or remote PowerShell to do its tasks. 

## Inventoty file `hosts`
```ini
[dev]
devsrv.me

[prod]
prodsrv.net
```

Ini-file example
```ini
mail.example.com

[webservers]
foo.example.com
bar.example.com

[dbservers]
one.example.com
two.example.com
three.example.com
```

YAML version
```yaml
all:
  hosts:
    mail.example.com:
  children:
    webservers:
      hosts:
        foo.example.com:
        bar.example.com:
    dbservers:
      hosts:
        one.example.com:
        two.example.com:
        three.example.com:
```

## Raw command when python not installed
Raw module will be used - https://docs.ansible.com/ansible/latest/modules/raw_module.html

```bash
ansible dev -i hosts -u root -m raw -a /bin/date
```

Response
```
devsrv.me | SUCCESS | rc=0 >>
Sun Jun  9 10:39:32 UTC 2019
Shared connection to devsrv.me closed.
```

### Raw command when python installed
```bash
ansible all -i hosts -u root -a "/bin/date"
```

Response
```
devsrv.me | SUCCESS | rc=0 >>
Sun Jun  9 11:08:29 UTC 2019

prodsrv.net | FAILED! => {
    "changed": false, 
    "module_stderr": "Shared connection to prodsrv.net closed.\r\n", 
    "module_stdout": "/bin/sh: 1: /usr/bin/python: not found\r\n", 
    "msg": "MODULE FAILURE", 
    "rc": 127
}
```

### Install python via raw command
```
ansible dev -i hosts -u root -m raw -a "apt -y install python"
```

## Modules
```
ansible dev -i hosts -u root -m git -a "repo=https://github.com/arusland/media-ext-bot.git dest=/root/media-bot"
```

## Playbooks
Playbooks are expressed in YAML format and have a minimum of syntax, which intentionally tries to not be a programming language or script, but rather a model of a configuration or a process.

Each playbook is composed of one or more ‘plays’ in a list.

The goal of a play is to map a group of hosts to some well defined roles, represented by things ansible calls tasks. At a basic level, a task is nothing more than a call to an ansible module

For starters, here’s a playbook, verify-apache.yml that contains just one play:
```yaml
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum:
      name: httpd
      state: latest
  - name: write the apache config file
    template:
      src: /srv/httpd.j2
      dest: /etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service:
      name: httpd
      state: started
  handlers:
    - name: restart apache
      service:
        name: httpd
        state: restarted
```
Playbooks can contain multiple plays. You may have a playbook that targets first the web servers, and then the database servers.

Remote users can also be defined per task:
```yaml
---
- hosts: webservers
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: yourname
```

You can also login as you, and then become a user different than root:
```yaml
---
- hosts: webservers
  remote_user: yourname
  become: yes
  become_user: postgres
```

### Tasks list
Each play contains a list of tasks. Tasks are executed in order, one at a time, against all machines matched by the host pattern, before moving on to the next task. It is important to understand that, within a play, all hosts are going to get the same task directives. It is the purpose of a play to map a selection of hosts to tasks.

### Handlers: Running Operations On Change

As we’ve mentioned, modules should be idempotent and can relay when they have made a change on the remote system. Playbooks recognize this and have a basic event system that can be used to respond to change.

These ‘notify’ actions are triggered at the end of each block of tasks in a play, and will only be triggered once even if notified by multiple different tasks.

For instance, multiple resources may indicate that apache needs to be restarted because they have changed a config file, but apache will only be bounced once to avoid unnecessary restarts.

Here’s an example of restarting two services when the contents of a file change, but only if the file changes:

```yaml
- name: template configuration file
  template:
    src: template.j2
    dest: /etc/foo.conf
  notify:
     - restart memcached
     - restart apache

handlers:
    - name: restart memcached
      service:
        name: memcached
        state: restarted
    - name: restart apache
      service:
        name: apache
        state: restarted
```

As of Ansible 2.2, handlers can also “listen” to generic topics, and tasks can notify those topics as follows:

```yaml
handlers:
    - name: restart memcached
      service:
        name: memcached
        state: restarted
      listen: "restart web services"
    - name: restart apache
      service:
        name: apache
        state: restarted
      listen: "restart web services"

tasks:
    - name: restart everything
      command: echo "this task will restart the web services"
      notify: "restart web services"
```

How do you run a playbook? It’s simple
```bash
ansible-playbook playbook.yml
```

## Roles
Roles are ways of automatically loading certain vars_files, tasks, and handlers based on a known file structure. Grouping content by roles also allows easy sharing of roles with other users.

### Role Directory Structure

Example project structure:
```
site.yml
webservers.yml
fooservers.yml
roles/
   common/
     tasks/
     handlers/
     files/
     templates/
     vars/
     defaults/
     meta/
   webservers/
     tasks/
     defaults/
     meta/
```
Roles expect files to be in certain directory names. Roles must include at least one of these directories, however it is perfectly fine to exclude any which are not being used. When in use, each directory must contain a main.yml file, which contains the relevant content:

* **tasks** - contains the main list of tasks to be executed by the role.
* **handlers** - contains handlers, which may be used by this role or even anywhere outside this role.
* **defaults** - default variables for the role (see Using Variables for more information).
* **vars** - other variables for the role (see Using Variables for more information).
* **files** - contains files which can be deployed via this role.
* **templates** - contains templates which can be deployed via this role.
* **meta** - defines some meta data for this role. See below for more details.

Other YAML files may be included in certain directories. For example, it is common practice to have platform-specific tasks included from the tasks/main.yml file:

```yaml
# roles/example/tasks/main.yml
- name: added in 2.4, previously you used 'include'
  import_tasks: redhat.yml
  when: ansible_facts['os_family']|lower == 'redhat'
- import_tasks: debian.yml
  when: ansible_facts['os_family']|lower == 'debian'

# roles/example/tasks/redhat.yml
- yum:
    name: "httpd"
    state: present

# roles/example/tasks/debian.yml
- apt:
    name: "apache2"
    state: present
```

## Ansible Galaxy
Ansible Galaxy is a free site for finding, downloading, rating, and reviewing all kinds of community developed Ansible roles and can be a great way to get a jumpstart on your automation projects.

The client ansible-galaxy is included in Ansible. The Galaxy client allows you to download roles from Ansible Galaxy, and also provides an excellent default framework for creating your own roles.

## Links
* Ansible Docs - https://docs.ansible.com/ansible/latest/index.html
* Quick start video https://www.ansible.com/resources/videos/quick-start-video
* Module index - https://docs.ansible.com/ansible/latest/modules/modules_by_category.html
