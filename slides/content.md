# Catalyst <!-- .element: class="catalyst-logo" -->

## Ansible for Samba
Presented by <!-- .element: class="small-text" --> [Joe Guo](https://github.com/guoqiao) <!-- .element: class="small-text" -->



## YAML
- a super set of JSON
- case sensitive
- use indent, space only, no tab allowed
- comment with `#`


### Data Types
- Scalars
- List
- Map


### Scalars

```
ENV_NAME: traffic
OS_NETWORK_NAME: "traffic-network"
OS_AUTO_FLOATING_IP: yes  # no
RUNNER_LOCKED: false  # true
RUNNER_PAUSED: False  # True
RUNNER_LIMIT: 40
```


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
  name: default
  CI_SERVER_URL: "https://gitlab.com"
  REGISTRATION_TOKEN: "XXXXXX"
  RUNNER_TAG_LIST: "docker,private"

primary_dc: {name: "dc0", ip: "10.10.10.10"}
```



## Jinja2

    <ul>
    {% for user in users %}
      <li><a href="{{ user.url }}">{{ user.username|lower }}</a></li>
    {% endfor %}
    </ul>



## Install

pip:

    pip install ansible

Install source:

    git clone https://github.com/ansible/ansible.git
    cd ansible
    pip install -I -e .


## Ansible Basics

- General purpose automation tool
- Agentless
- Python and SSH
- Push not poll


## However Ansible works
- define hosts in inventory
- define connection parameters and variables
- define tasks for selected hosts in play
- connect to hosts in parallel(ssh/winrm/docker/local)
- push scripts to hosts(module)
- run module scripts with python/powershell
- fetch results as json/yaml


## Inventory

    [ubuntu]
    server0 ansible_host=202.49.242.50 ansible_user=ubuntu ansible_python_interpreter=/usr/bin/python3
    server1 ansible_host=202.49.242.51 ansible_user=ubuntu ansible_python_interpreter=/usr/bin/python3


[Behavior Inventory Parameters](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#list-of-behavioral-inventory-parameters)

    ansible_connection: ssh
    ansible_python_interpreter: /usr/bin/python
    ansible_user: $USER
    ansible_become: no
    ansible_become_user: root
    ansible_ssh_port: 22
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ...


### Inventory path

    /etc/ansible/hosts  # default
    ./playbook.yml -i hosts.ini  # cli option
    inventory = hosts.ini # ansible.cfg


### group vars in inventory

    # hosts.ini

    [ubuntu]
    dc0 ansible_host=202.49.242.50
    dc1 ansible_host=202.49.242.51

    [ubuntu:vars]
    ansible_user=ubuntu
    ansible_python_interpreter=/usr/bin/env python3


### group vars in yml

    # hosts.ini
    [ubuntu]
    dc0 ansible_host=202.49.242.50
    dc1 ansible_host=202.49.242.51

    # group_vars/ubuntu.yml:
    ansible_user: ubuntu
    ansible_python_interpreter: /usr/bin/env python3



## Ad hoc commands

    ansible -i hosts.ini ubuntu -m ping
    ansible -i hosts.ini ubuntu -m setup  # gather facts
    ansible -i hosts.ini ubuntu -m apt -a "name=git"  # module with args
    ansible localhost -a "uname -a"  # default module: command



## playbook

    #!/usr/bin/env ansible-playbook
    ---
    - name: this is a play
      hosts: ubuntu  # host or group
      tasks:
        - name: this is a task
          apt:  # <- module, normally written by python
            become: yes
            update_cache: yes
            name:
              - python3-dev
              - python3-pip


run a playbook:

    ansible-playbook -i hosts.ini playbook.yml


run yml file directly

add shebang:

    #!/usr/bin/env ansible-playbook

make yml executable

    chmod a+x playbook.yml

then run it:

    ./playbook.yml -i hosts.ini


## output color


## template


## when(condition)


## loop


## register


## debug (debug, register, step)


## dynamic inventory


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
