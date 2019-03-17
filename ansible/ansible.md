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







