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


## Ansible Install

    pip install ansible

    # Install source
    git clone https://github.com/ansible/ansible.git
    cd ansible
    pip install -I -e .


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

    autobuild/inventory/openstack_inventory.py
    autobuild/inventory/rax.py
    vagrant_inventory.py
    docker_inventory.py


### Inventory path

    /etc/ansible/hosts  # default
    ./playbook.yml -i hosts.ini  # cli option
    inventory = ./inventory/  # ansible.cfg as dir
    ANSIBLE_INVENTORY=$HOME/hosts.ini  # env vars



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
- group_vars: `all.yml/vagrant.yml`
- vars_file: `include_vars`
- vault: vault.yml
- role: `defaults/main.yml`, `vars/main.yml`
- task: vars
- play: vars, vars_files
- cli option: `-e ENV_NAME=gitlab-runner`
- facts
- set_fact
- register


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

    myrole/
      defaults/
      tasks/
      vars/
      handlers/
      templates/
      files/
      meta/













## Module examples

## ansible.cfg


## tests


## role ang ansible-galaxy


## openstack role

## traffic replay

## gitlab-runner


## samba-common|samba-dc|samba-traffic-runner


## Title <!-- .slide: class="image-slide" -->
![placeholder image](img/800x400.gif "Placeholder image")



### List title
* List item ![placeholder image](img/300x400.gif "Placeholder image") <!-- .element: class="img-right" -->
* List item
* List item



### Fragmented list
* List item <!-- .element: class="fragment" -->
* List item <!-- .element: class="fragment" -->
* List item <!-- .element: class="fragment" -->
![placeholder image](img/500x200.gif "Placeholder image") <!-- .element: class="img-centered" -->



## Title (press down to see vertical slides) <!-- .slide: class="title-slide banner-3" --> <!-- .element: class="yellow" -->


### Paragraph
Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris at scelerisque eros. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Aliquam placerat posuere nibh vel dapibus.


### Blockquote
> Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris at scelerisque eros. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Aliquam placerat posuere nibh vel dapibus.


### Nested lists
+ One
+ Two
+ Three
	- Nested One
	- Nested Two



### Title
Some text

	function linkify( selector ) {
		if( supports3DTransforms ) {

			var nodes = document.querySelectorAll( selector );

			for( var i = 0, len = nodes.length; i < len; i++ ) {
				var node = nodes[i];

				if( !node.className ) {
					node.className += ' roll';
				}
			}
		}
	}



### Image backgrounds <!-- .slide: data-background="img/800x600.gif" -->



![freedom to innovate](css/theme/images/freedom-to-innovate.svg "freedom to innovate") <!-- .slide: class="image-slide" --> <!-- .element: class="cat-slogan-image" -->
