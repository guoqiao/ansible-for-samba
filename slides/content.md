# Catalyst <!-- .element: class="catalyst-logo" -->

## Ansible for Samba
Presented by <!-- .element: class="small-text" --> [Joe Guo](https://github.com/guoqiao) <!-- .element: class="small-text" -->



## YAML

### Data Types
- Scalars
- List/Sequence
- Map/Object/Dictionary


### Scalars

    ENV_NAME: traffic  # quotes are optional for str
    OS_NETWORK_NAME: "traffic-network"
    RUNNER_LIMIT: 40  # int
    LATENCY: 5.0  # float
    SAMBA_PASSWORD: "123456"
    OS_AUTO_FLOATING_IP: yes  # bool, no quotes


#### Boolean in YAML

     y|Y|yes|Yes|YES|n|N|no|No|NO
    |true|True|TRUE|false|False|FALSE
    |on|On|ON|off|Off|OFF


### List
```YAML
OS_NAMESERVERS:
  - 202.78.247.197
  - 202.78.247.198
  - 202.78.247.199
OS_SECURITY_GROUPS: ["default", "domain"]
```


### Map
```
gitlab_runner_register_target:
  CI_SERVER_URL: "https://gitlab.com"
  REGISTRATION_TOKEN: "XXXXXX"
  RUNNER_TAG_LIST: "docker,private"

OS_SERVER: {name: "dc0", flavor: "c1.c2r16"}
```


### Combined

    OS_SERVERS:
      - name: dc0
        flavor: c1.c2r8
        groups:
          - samba-common
          - samba-dc
      - name: dc1
        flavor: c1.c2r8
        groups:
          - samba-common
          - samba-dc
      - name: runner
        flavor: c1.c4r16
        groups:
          - samba-common
          - samba-traffic-runner



## Jinja2

    <ul>
    {% for user in users %}
      <li><a href="{{ user.url }}">{{ user.username|lower }}</a></li>
    {% endfor %}
    </ul>


### Jinja2 Filters

- [Builtin Jinja2 Filters](http://jinja.pocoo.org/docs/2.10/templates/#builtin-filters)
- [Ansible Jinja2 Filters](https://ansible-docs.readthedocs.io/zh/stable-2.0/rst/playbooks_filters.html#jinja2-filters)



## Ansible Basics

- General purpose automation tool
- Agentless
- Python and SSH
- Push not poll


## How Ansible works
- define hosts in inventory
- define connection parameters and variables
- define tasks for selected hosts in play
- connect to hosts in parallel(ssh/winrm/docker/local)
- push scripts to hosts(module)
- run module scripts with python/shell/powershell
- fetch results as json/yaml


## Inventory

    [servers]
    server0 ansible_host=202.49.242.50 ansible_user=centos
    server1 ansible_host=202.49.242.51 ansible_user=ubuntu ansible_python_interpreter=/usr/bin/python3
    server2 ansible_host=202.49.242.52 ansible_user=root ansible_ssh_port=2222


[Behavior Inventory Parameters](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters)

    ansible_connection: ssh
    ansible_python_interpreter: /usr/bin/python
    ansible_user: $USER
    ansible_become: no
    ansible_become_user: root
    ansible_ssh_port: 22
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ...


### group vars in inventory

    # hosts.ini

    [vagrant]
    dc0 ansible_host=202.49.242.50
    dc1 ansible_host=202.49.242.51

    [vagrant:vars]
    ansible_user=vagrant
    ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key


### group vars in yml

    # hosts.ini
    [vagrant]
    dc0 ansible_host=202.49.242.50
    dc1 ansible_host=202.49.242.51

    # group_vars/vagrant.yml:
    ansible_user: vagrant
    ansible_ssh_private_key_file: ~/.vagrant.d/insecure_private_key


### Dynamic inventory

A script to return hosts via cli as json dynamicly

    autobuild/inventory/openstack_inventory.py --list --refresh
    autobuild/inventory/rax.py --list --refresh-cache
    vagrant_inventory.py --list
    docker_inventory.py --list


### Inventory path

    /etc/ansible/hosts  # default
    ./playbook.yml -i hosts.ini  # cli option
    inventory = ./inventory/  # ansible.cfg as dir
    ANSIBLE_INVENTORY=$HOME/hosts.ini  # env vars



## Ansible Install

    pip install ansible

    # Install source
    git clone https://github.com/ansible/ansible.git
    cd ansible
    pip install -I -e .


## Ad hoc commands

    > ansible ubuntu -m ping
    > ansible localhost -m setup  # gather facts
    > ansible ubuntu -m apt -a "name=git"  # module with args
    > ansible ubuntu -a "uname -a"  # default module: command



## playbook

    #!/usr/bin/env ansible-playbook
    ---
    - name: this is a play
      hosts: ubuntu  # host or group, pattern
      tasks:
        - name: this is a task
          become: yes
          apt:  # <- module, normally written by python
            update_cache: yes
            name:
              - python3-dev
              - python3-pip


run a playbook:

    ansible-playbook -i hosts.ini playbook.yml

    # add shebang: #!/usr/bin/env ansible-playbook
    chmod a+x playbook.yml
    ./playbook.yml -i hosts.ini



## Variables

    # example in ansible-role-openstack/defaults/main.yml
    ENV_NAME: "{{ lookup('env', 'USER') }}"
    OS_NETWORK_NAME: "{{ ENV_NAME }}-network"
    OS_ROUTER_NAME: "{{ ENV_NAME }}-router"
    OS_SUBNET_NAME: "{{ ENV_NAME }}-subnet"
    OS_SUBNET_CIDR: "10.0.0.0/24"
    OS_AUTO_FLOATING_IP: yes


## Variable sources

- inventory file: `hosts.ini`
- yml file: group_vars/group.yml, vars.yml
- play/task/role
- cli option: `-e ENV_NAME=gitlab-runner`
- facts/set_fact/register


## Vault

    > ansible-vault -h
    Usage: ansible-vault [create|decrypt|edit|encrypt|encrypt_string|rekey|view] [options] [vaultfile.yml]

    # autobuild/cloud-network/ansible.cfg
    [defaults]
    vault_password_file = ~/.vault_password_autobuild



## Task & Module


### when(condition)

    - name: apt install python3 dev
      become: yes
      apt:
        name: python3-dev
      when: 'ansible_pkg_mgr == "apt"'

    - name: yum install python3 devel
      become: yes
      yum:
        name: python3-devel
      when: 'ansible_pkg_mgr == "yum"'


### loop

    - name: create server instances
      os_server:
        name: "{{ item.name }}"
        flavor: "{{ item.flavor }}"
        ...
      loop: "{{ OS_SERVERS }}"


### register

    - name: check docker-machine
      command: which docker-machine
      register: which_result

    - name: install docker-machine
      become: yes
      get_url:
        url: "{{ docker_machine_install_url }}"
        dest: "/usr/local/bin/docker-machine"
        mode: 0555
      when: which_result.rc != 0


### tags

    - name: check docker-machine
      command: which docker-machine
      tags: docker-machine

    > ./playbook.yml --tags docker-machine
    > ./playbook.yml --t docker-machine

    - name: gater facts
      tags: always  # a special tag which will always run
      setup:

    - name: delete server
      tags: ['never', 'delete']  # a special tag only run when you ask for it
      os_server:
        name: dc0
        state: absent


### debug

    - debug: msg="The vaulue for my_var is: {{my_var}}"
    - debug: var=my_var
    - debug: var=vars

    ./playbook.yml -v
    ./playbook.yml -vvvv
    ./playbook.yml --step
    ./playbook.yml -m setup --limit dc0
    ./playbook.yml --start-at-task "check docker-machine"
    ./playbook.yml --tags docker-machine
    ./ansible-inventory -i hosts.ini


## template

    - name: render /etc/docker/daemon.json
      become: yes
      template:
        src: templates/daemon.json
        dest: /etc/docker/daemon.json
        mode: 0644
        force: no


### lineinfile

    - name: add LANG in /etc/locale.conf
      lineinfile:
        path: /etc/locale.conf
        regexp: '^LANG='
        line: 'LANG=en_US.utf8'
        create: yes



## Role

    example
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml



### [role: openstack](https://gitlab.com/catalyst-samba/ansible-role-openstack)

Ansible Role to setup infrastructure and servers in openstack cloud.


#### default

    - name: run role openstack
      hosts: localhost
      tasks:
        - name: setup infra and launch 1 ubuntu server
          include_role:
            name: openstack


#### tasks_from

    - name: run role openstack
      hosts: localhost
      vars:
        ENV_NAME: gitlab-runner
      tasks:
        - name: run openstack role for infra only
          include_role:
            name: openstack
            tasks_from: infra  # servers


#### vars_from

    - name: run role openstack
      hosts: localhost
      tasks:
        - name: run openstack role to launch windows instances
          include_role:
            name: openstack
            vars_from: windows
          vars:
            ENV_NAME: wintest
            OS_SERVERS:
              - name: win0
              - name: win1


#### large scale network

    - name: run role openstack
      hosts: localhost
      tasks:
        - name: run openstack role to create large number of servers in parallel
          include_role:
            name: openstack
          vars:
            ENV_NAME: indeed
            OS_SERVERS: "{{ range(50)|list }}"



### [role: rackspace](https://gitlab.com/catalyst-samba/ansible-role-rackspace)

    - name: run role rackspace
      hosts: localhost
      tasks:
        - name: run rackspace role to launch server
          include_role:
            name: rackspace
          vars:
            RS_SERVER_NAME: rax0
            RS_SERVER_GROUPS: ['gitlab-runner', 'rackspace']
            RS_FLAVOR_ID: 4


### samba roles

- [samba-common](https://gitlab.com/catalyst-samba/ansible-role-samba-common)
- [samba-dc](https://gitlab.com/catalyst-samba/ansible-role-samba-dc)
- [samba-traffic-runner](https://gitlab.com/catalyst-samba/ansible-role-samba-traffic-runner)


### windows roles

- [windows-common](https://gitlab.com/catalyst-samba/samba-cloud-autobuild/roles/windows-common)
- [windows-dc](https://gitlab.com/catalyst-samba/samba-cloud-autobuild/roles/windows-dc)
- [windows-wpts](https://gitlab.com/catalyst-samba/samba-cloud-autobuild/roles/windows-wpts)


### other roles
- [locale](https://gitlab.com/catalyst-samba/ansible-role-locale)
- [xbuild](https://gitlab.com/catalyst-samba/ansible-role-xbuild)
- [gitlab-runner](https://gitlab.com/catalyst-samba/ansible-role-gitlab-runner)



## playbook examples

refer to [autobuild/cloud-network](https://gitlab.com/catalyst-samba/samba-cloud-autobuild/tree/master/cloud-network)


### cloud-network/traffic-replay.yml

    # ./traffic-replay.yml -e ENV_NAME=traffic -e primary_dc_name=traffic-dc0

    - name: create servers in openstack cloud
      hosts: localhost
      tasks:
        - name: include role openstack
          include_role:
            name: openstack
            vars_from: ubuntu
          vars:
            OS_SERVERS:
              - name: "{{ENV_NAME}}-dc0"
                flavor: c1.c2r8
                groups:
                  - samba-common
                  - samba-dc
              - name: "{{ENV_NAME}}-runner"
                flavor: c1.c4r16
                groups:
                  - samba-common
                  - samba-traffic-runner

    - name: include role samba-common
      # hosts pattern, use meta groups
      hosts: "{{ENV_NAME}}:&samba-common"
      tasks:
        - name: include role samba-common
          include_role:
            name: samba-common

    - name: include role samba-dc
      hosts: "{{ENV_NAME}}:&samba-dc"
      tasks:
        - name: include role samba-dc
          include_role:
            name: samba-dc

    - name: include role samba-traffic-runner
      hosts: "{{ENV_NAME}}:&samba-traffic-runner"
      tasks:
        - name: include role samba-traffic-runner
          include_role:
            name: samba-traffic-runner
          vars:
            run_cleanup: yes
            run_replay: yes
            run_summary: yes


### windows-domain.yml

    - name: create 2 windows instances
      hosts: localhost
      tasks:
        - name: include role openstack
          include_role:
            name: openstack
            vars_from: windows
          vars:
            ENV_NAME: windomain
            OS_SERVERS: "{{range(2)|list}}"
            OS_META_GROUPS:
              - windows-common
              - windows-dc

    - name: apply role windows-common
      hosts: "windomain:&windows-common"
      vars:
        ENV_NAME: windomain
        primary_dc_name: windomain-win0
      roles:
        - windows-common
        - windows-dc


### gitlab-ci/openstack.yml

    - name: launch instance
      hosts: localhost
      tasks:
        - name: include role openstack
          include_role:
            name: openstack
          vars:
            OS_SERVER_NAME: "{{ RUNNER_NAME }}"

    - name: setup gitlab-runner
      hosts: "{{ RUNNER_NAME }}"
      tasks:
        - name: include role gitlab-runner
          include_role:
            name: gitlab-runner
            vars_from: docker-machine/openstack
          vars:
            gitlab_runner_register_targets: "{{vault_gitlab_runner_register_targets_openstack}}"

    # > ./openstack.yml -v -e ENV_NAME=gitlab-runner -e RUNNER_NAME=gitlab-runner-openstack-$(date +%m%d)


### gitlab-ci/rackspace.yml

    - name: launch instance
      hosts: localhost
      tasks:
        - name: include role rackspace
          include_role:
            name: rackspace
          vars:
            RS_SERVER_NAME: "{{ RUNNER_NAME }}"

    - name: setup gitlab-runner
      hosts: "{{ RUNNER_NAME }}"
      tasks:
        - name: include role gitlab-runner
          include_role:
            name: gitlab-runner
            vars_from: docker-machine/rackspace
          vars:
            gitlab_runner_register_targets: "{{vault_gitlab_runner_register_targets_rackspace}}"

    # > ./rackspace.yml -v -e ENV_NAME=gitlab-runner -e RUNNER_NAME=gitlab-runner-rackspace-$(date +%m%d)



## Q & A
![freedom to innovate](css/theme/images/freedom-to-innovate.svg "freedom to innovate") <!-- .slide: class="image-slide" --> <!-- .element: class="cat-slogan-image" -->
