# Ansible

General notes to start using Ansible.

## Installation

Install Ansible using `apt` for a system wide installation. Alternatively Ansible can be installed via `pip`.

```bash
$ sudo apt update
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt install ansible
```

## Configuration

### Sudoers

Add the Ansible user to the host. In this case, the user will be `evan`.

```bash
$ sudo adduser evan
$ sudo adduser evan sudo # adds the user to the sudo group
```

Running `visudo -f <file>` or `visudo --file=<file>` will allow you to specify an alternate sudoers file rather than the default `/etc/sudoers`.

```bash
$ sudo EDITOR=vim visudo -f /etc/sudoers.d/90-init-users
```

```bash
# File 90-init-users

# User rules for the Ansible user
evan ALL=(ALL) NOPASSWD:ALL
```

### SSH

The SSH software Ansible uses is the underlying OpenSSH package `openssh-server` used in most operating systems.

* Ansible expects to be able to connect to systems in the inventory via SSH
* SSH can be configured using a configuration file with a default location of `/home/evan/.ssh/config`

```bash
$ ssh-keygen # from the controller
$ ssh-copy-id evan@<host ip>
```

#### Going Beyond SSH

Ansible supports other methods of controlling systems.

* Local (directly on localhost)
* WinRM (Windows Remote Management)
* Docker (`docker exec`)

## Test Functionality

```bash
$ ansible all -i <host ip>, -m ping # specifying a single host must end in a comma
$ ansible all -i <host ip>,<host ip> -m command --args 'uptime'
```

## Inventory File

`ansible-playbook` lets us place an Ansible configuration into files so we can reliably run the same set of commands.

* Controls what systems are used for `ansible-playbook`
* Allows for per-system configuration
* `.ini` file format

The default inventory file location is `/etc/ansible/hosts`.

### Groups

```ini
[database]
database01

[dotcms]
dotcms01
dotcms02
dotcms03

[ubuntu:childern] # add an entire group to the ubuntu group
database
dotcms

[ubuntu:vars] # variables specific to the ubuntu group
ansible_user=evan

[balancer]
balancer01 ansible_user=root
```

## Playbooks

Every `ansible-playbook` command starts with a playbook YAML file.

* Each command Ansible applies to a remote system is called a *task*

  ```yaml
  - name: install jre
    package: name=openjdk-8-jre state=installed
  ```

* A *play* groups tasks together

  ```yaml
  - hosts: all
    become: true
    tasks:
      - name: install jre
        package: name=openjdk-8-jre state=installed
      - name: group
        group:
          name: dotcms
          state: present
  ```

* A *playbook* groups plays together

  ```yaml
  ---
  # play 1
  - hosts: all # all covers all of the systems in the inventory
    become: true # boolean value
    tasks: # YAML list of tasks (invididual tasks are dictionaries)
      - name: install jre
        package: name=openjdk-8-jre state=installed
      - name: group
        group:
          name: dotcms
          state: present
  # play 2
  - hosts: balancer # plain value (scalar) containing a string
    become: true
    tasks:
      - name: install haproxy
        package: name=haproxy state=installed
  ```

  It's good practice to leave plays at the group level even if there's only one system

## YAML

* Plain values (scalars)

  ```yaml
  key: string value
  key: "string value"
  key: 123 # numeric value
  key: 123.45 # numeric value
  become: true # boolean
  ```

* Data structures

  ```yaml
  list:
    - lots
    - of
    - things
  dictionary: # map / hash
    data: here you go
    howmany: 789
    good: yes
  ```

## Tasks

* A task is one individual *thing* Ansible should do

  ```yaml
  - name: install jre # friendly title (optional)
    package: name=openjdk-8-jre state=installed # Ansible module to run with parameters
  ```

### Long and Short Form

* Long

  ```yaml
  - service:
      name: dotcms
      state: started
      enabled: yes
  ```

* Short

  ```yaml
  - service: name=dotcms state=started enabled=yes
  ```

## Handlers

Handlers are just like tasks, but they only run if notified (conditional).

```yaml
- name: configure
  copy:
    src: postgresql.conf
    dest: /etc/postgresql/{{ postgresql_version }}/main/postgresql.conf
    owner: postgres
    group: postgres
    mode: 0644
  notify:
    - restart postgresql # only restart PostgreSQL if the configuration file has changed
```

In the event that Ansible changed `postgresql.conf` on the remote system, the `restart postgresql` handler will be notified.

```yaml
- name: restart postgresql
  service: name=postgresql state=restarted
```

### Flushing Handlers

* Ansible saves handlers until all tasks are finished

* Sometimes we need handlers to run before another task

  * e.g. restart the database before adding users

* Ansible has a *meta* command `flush_handlers` to do this (runs immediately)

  ```yaml
  - meta: flush_handers
  ```

## Ansible Become

*Become* is Ansible's generic term for privilege escalation. Task level overrides play level (all tasks in play), and play level overrides host level (all plays for host).

## Controlling Playbook Runs

```bash
$ ansible-playbook -i inventory playbook.yaml -l balancer # further limit selected hosts
$ ansible-playbook -i inventory playbook.yaml --tags service # limit tasks to run by given tags
```

## Applying Roles to Multiple Systems

Roles help us build a reusable library of automation. Playbooks quickly get large and hard to maintain. We want to break up our tasks into modules so we can capture and reuse them.

### Roles Capture Important Information

* Tasks - Actions for automatic install and config
* Files - Configuration files, including customizable templates
* Variables - Separate configuration choices to make them easier to see

### Role Directory Components

* defaults - default configuration files

* files - files to deploy to the remote system

* handlers - conditional tasks that run if notified

* meta - role metadata (e.g. dependencies)

* tasks - main entry point, actions to complete

* templates - customizable files to deploy

* vars - high priority configuration variables

## Applying Commonly Used Modules

### Package Modules

* OS packages - This is for installing OS packages and it's smart enough to know when it should use `yum` or `apt`

  ```yaml
  - name: install jre
    package: name=openjdk-8-jre state=installed
  ```

  In this case `installed` means it wants to make sure that some version of that package is installed

* Package repositories

  ```yaml
  - name: nodesource repository
    yum_repository:
      name: nodesource
      description: Nodesource Yum Repo
      baseurl: https://rpm.nodesource.com/pub_8.x/el/7/$basearch/
  ```

  ```yaml
  - name: nodesource repository
    apt_repository:
      repo: deb https://deb.nodesource.com/node_8.x xenial main
      state: present
  ```

* Python packages

  ```yaml
  - name: tensorflow
    pip:
      name: tensorflow
      virtualenv: /opt/tensorflow
  ```

  The `pip` module can install libraries into the system or install into a virtual environment

* Rubygems

  ```yaml
  - name: rake
    gem:
      name: rake
      state: present
  ```

### File Modules

Ansible is smart enough to know that the `copy` module should look into the `files` subdirectory of the role. The `template` module should look in the `templates` subdirectory of the role.

* Copy file to remote system

  ```yaml
  - name: configure
    copy:
      src: postgresql.conf
      dest: /etc/postgresql/9.5/main/postgresql.conf
      owner: postgres
      group: postgres
      mode: 0644
    notify:
      - restart postgresql
  ```

* Copy file, but update based on variables - Ansible will run the template process to see what the content of the file will be once the variables are populated

  ```yaml
  - name: context xml
    template:
      src: context.xml
      dest: "opt/dotcms/dotserver/tomcat-8.0.18/webapps/ROOT/META-INF/context.xml"
      owner: "dotcms"
      group: "dotcms"
      mode: 0644
  ```

* Create/delete files/directories

  ```yaml
  - name: installation directory
    file:
      name: "/opt/dotcms"
      state: directory
      owner: "dotcms"
      group: "dotcms"
      mode: 0755
  ```

* Update a line in a file

  ```yaml
  - name: ensure localhost in hosts
    lineinfile:
      path: /etc/hosts
      regexp: '^127\.0\.0\.1'
      line: '127.0.0.1 localhost'
      owner: root
      group: root
      mode: 0644
  ```

* Extract from from compressed files

  ```yaml
  - name: install
    unarchive:
      src: "/opt/dotcms.tar.gz"
      dest: "/opt/dotcms"
      owner: "dotcms"
      group: "dotcms"
      creates: "/opt/dotcms/bin/startup.sh"
  ```

### System Modules

* Ensure services are started

    ```yaml
    - name: service
      service: name=dotcms state=started enabled=yes
    ```

    The Ansible service module is smart enough to deal with `systemd` and `init.d` services

* Create users and groups

    ```yaml
    - name: group
      group: name=dotcms state=present
    ```

    ```yaml
    - name: user
      user:
        name: "dotcms"
        group: "dotcms"
        home: "/opt/dotcms"
        state: present
        system: yes
    ```

* Create `cron` jobs

    ```yaml
    - name: schedule yum autoupdate
      cron:
        name: yum autoupdate
        weekday: 2
        minute: 0
        hour: 12
        user: root
        job: "YUMINTERACTIVE: 0 /usr/sbin/yum-autoupdate"
    ```

* Security enhanced Linux extensions (SELinux)

    ```yaml
    - name: allow httpd to make network connections
      seboolean:
        name: httpd_can_network_connect
        state: yes
        presistent: yes
    ```

## Ansible Galaxy

Ansible Galaxy refers to the [Galaxy website](https://galaxy.ansible.com/) where users can share roles, and to a command line tool `ansible-galaxy` for installing, creating, and managing roles. This is just a listing and search service, the actual role itself is downloaded from [GitHub](https://github.com/).

### Install Roles

```bash
$ ansible-galaxy install geerlingguy.java # installs into ~/.ansible/roles
```

This can have the result of having a more complicated role than what we would normally write ourselves. We may find it to be easier to write a role that's narrowly tailored to the work that we want, or to find a role and then figure out how to customize it so we can use it correctly.

The information in the `meta` subdirectory is used to display information on the role's Galaxy webpage.

### Where Ansible Looks for Roles

Ansible will use the `ANSIBLE_ROLES_PATH` environment variable, a colon separated list of paths that tell Ansible where to look for roles. This has a default value of `~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles`.

## Using Role Handlers, Files, and Templates

### Rules for Handlers

* Only run if *notified*

* Run at the *end of play* (unless flushed)

* Match by *name*

* Roles can reference own or *dependent* handlers

### Rules for Files and Templates

* Should usually be kept in own role

* Subdirectories inside files and templates are OK

* Search for relative paths (Ansible will look in different places in a particular order):

  |             Copy              |             Template              |
  | :---------------------------: | :-------------------------------: |
  |   `<role dir>/files/<path>`   |   `<role dir>/templates/<path>`   |
  |   `<role dir>/tasks/<path>`   |     `<role dir>/tasks/<path>`     |
  |      `<role dir>/<path>`      |        `<role dir>/<path>`        |
  | `<playbook dir>/files/<path>` | `<playbook dir>/templates/<path>` |
  |    `<playbook dir>/<path>`    |      `<playbook dir>/<path>`      |

## Role Dependencies

Notice the playbook below only lists the `tomcat` role. The dependencies for `tomcat` will be listed in `roles/tomcat/meta/main.yml`. The `meta` subdirectory is used for providing metadata or information about the role.

```yaml
# File playbook.yml
---
- hosts: all
  become: true
  roles:
    - tomcat
```

Examples:

* In the `meta` subdirectory a `main.yml` file is used to list another role that `tomcat` is dependent on

  ```yaml
  # File roles/tomcat/meta/main.yml
  ---
  dependencies:
    - java # the java role will run prior to tomcat
  ```

* Provide a parameter to override a default variable `java_package` defined in `roles/java/defaults/main.yml`

  ```yaml
  # File roles/tomcat/meta/main.yml
  ---
  dependencies:
    - { role: java, java_package: openjdk-9-jre }
  ```

* In this case the parameters are different so the role we be ran twice

  ```yaml
  # File roles/tomcat/meta/main.yml
  ---
  dependencies:
    - { role: java, java_package: openjdk-8-jre }
    - { role: java, java_package: openjdk-9-jre }
  ```

## Using the Template Library

The template module will take a source file and add content to the defined variables. The resulting file will be compared to the file on the remote system to determine if it needs to be updated.

When Ansible creates templates it relies on a popular Python library called [Jinja2](http://jinja.pocoo.org/). This library does the templating work while Ansible is responsible for updating the file and file attributes on the remote system.

### Template Flow Control

```yaml
# File roles/haproxy/defaults/main.yml
---
haproxy_listen_address: 0.0.0.0
haproxy_listen_port: 80
haproxy_stats: True
```

```jinja2
# File roles/haproxy/templates/haproxy.cfg
...
frontend haproxy
    {# variable replacement #}
    bind {{ haproxy_listen_address }}:{{ haproxy_listen_port }}
    mode http
    default_backend dotcms
{# control statement #}
{% if haproxy_stats %}
    stats enable
    stats uri /haproxy?stats
    stats realm HAProxy Statistics
    stats auth admin:admin
{% endif %}
    balance roundrobin
    option httpclose
    option forwardfor
...
```

### Repeated Configuration Content

Examples:

* Using a list

   ```yaml
   # File roles/haproxy/defaults/main.yml
   ...
   haproxy_backends:
     - 'server dotcms01 172.31.0.21:8080 check' # hyphen indicates that this is a list
     - 'server dotcms02 172.31.0.22:8080 check'
   ...
   ```

   ```jinja2
   # File roles/haproxy/templates/haproxy.cfg
   ...
   backend dotcms
   {% for backend in haproxy_backends %}
       {{ backend }}
   {% endfor %}
   ...
   ```

* Using a dictionary

   ```yaml
   # File roles/haproxy/defaults/main.yml
   ...
   haproxy_backends:
     dotcms01:
       ip: 172.31.0.21
     dotcms02:
       ip: 172.31.0.22
   ...
   ```

   ```jinja2
   # File roles/haproxy/templates/haproxy.cfg
   ...
   backend dotcms
   {% for key, value in haproxy_backends.iteritems() %}
       server {{ key }} {{ value.ip }}:8080 check
   {% endfor %}
   ...
   ```

* Using a macro

   ```jinja2
   # File roles/haproxy/templates/backend.j2

   {% macro backend(name, ip, port=8080) -%}
       server {{ name }} {{ ip }}:{{ port }} check
   {%- endmacro %}
   ```

     ```jinja2
   # File roles/haproxy/templates/haproxy.cfg
   ...
   {# needs to be imported if using a separate file #}
   {% import 'backend.j2' as backend %}
   backend dotcms
   {% for key, value in haproxy_backends.iteritems() %}
   backend(key, value.ip)
   {% endfor %}
   ...
     ```

### Using Defaults and Filters

Examples:

* To remove the requirement of a variable being defined the `default` filter can be used to supply a default value

  ```jinja2
  # File roles/haproxy/templates/haproxy.cfg
  ...
  frontend haproxy
      {# variable replacement #}
      bind {{ haproxy_listen_address }}:{{ haproxy_listen_port|default('80') }}
      mode http
      default_backend dotcms
  {# control statement #}
  {% if haproxy_stats|default(True) %}
      stats enable
      stats uri /haproxy?stats
      stats realm HAProxy Statistics
      stats auth admin:admin
  {% endif %}
  ...
  ```

* Use `join` to concatenate strings with an optional parameter

  ```yaml
  # File roles/<role>/defaults/main.yml
  ...
  servers:
    - server01
    - server02
    - server03
  ...
  ```

  ```jinja2
  # File roles/<role>/templates/<template>
  ...
  {{ servers|join(',') }}
  {# result server01,server02,server03 #}
  ...
  ```

* Use `map` to apply a filter on a sequence of objects or look up an attribute to return a list of values

  ```yaml
  # File roles/<role>/defaults/main.yml
  ...
  haproxy_backends:
    dotcms01:
      ip: 172.31.0.21
    dotcms02:
      ip: 172.31.0.22
  ...
  ```

  ```jinja2
  # File roles/<role>/templates/<template>
  ...
  {{ haproxy_backends.values()|map(attribute='ip')|join(',') }}
  {# result 172.31.0.21,172.31.0.22 #}
  ...
  ```
