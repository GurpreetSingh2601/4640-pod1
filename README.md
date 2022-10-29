# Create 2 droplets on Digital Ocean
#Note - It is recommended to use a single ssh key for both VMs for easy login (Not for production use)
* Ubuntu 22.04
* Rocky Linux

##### SSH into both VMs to test that you can ping and connect to them
```bash
ssh root@<vm public ipv4 address>
```
# Configuring Ansible
### 1. Create an ansible configuration file
```bash
vim ansible.cfg
```
##### which contains the following:
```bash
[defaults]
host_key_checking = False
inventory = inventory

[inventory]
enabled_plugins = community.digitalocean.digitalocean
```
### 2. Generate a new API token 
### Note- We are creating API token in digital ocean, after creating it we need to copy and save it in a .profile file to export it and use it

```bash
export DO_API_TOKEN=<Digital ocean API token>
```
### After creating the .profile file we will source it to make our token reusable as a variable

```bash
source .profile
```

### 3. Create a new inventory directory with a host file
```bash
touch ./inventory/host.digital_ocean.yml
```

##### which contains the following:
### Note - To use the following configurations your VM's group name should be webserver and they have tag named "web" attached to them
```bash
plugin: community.digitalocean.digitalocean
attributes:
  - id
  - name
  - memory
  - vcpus
  - disk
  - size
  - image
  - networks
  - volume_ids
  - tags
  - region
groups: 
  webserver: "'web' in (do_tags)"
compose:
  ansible_host: do_networks.v4 | selectattr('type','eq','public')
    | map(attribute='ip_address') | first

```
##### To test the ansible set up 
```bash
ansible -m ping -u root all
```

##### checking the ansible inventory graph
```bash
$ ansible-inventory --graph
```
# Creating Ansible playbook

## Create User Playbook
Creates a new regular user on both the Ubuntu and Rocky Linux VMs with the following:
1. A home directory in /home
2. A bash as their login shell 
3. The ability to use sudo

```bash
vim create_user.yml
```

```YAML
---

- name: creating a new user
  hosts: all
  tasks:
    - name: add a new user in ubuntu
      become: true
      user:
        name: week4user
        password: "{{ 'week4user' | password_hash('sha512') }}"
        groups: "sudo"
        state: present
        shell: "/bin/bash"
        createhome: true
        home: /home
        append: yes

      when: ansible_distribution in ["Ubuntu"]

    - name: add a new user in Rocky Linux
      become: true
      user:
        name: week4user
        password: "{{ 'week4user' | password_hash('sha512') }}"
        append: yes
        create_home: true
        home: /home
        shell: "/bin/bash"
        groups: "wheel"

      when: ansible_distribution in ["Rocky"]

    - name: copy the ssh folder
      become: true
      copy:
        src: ~/.ssh
        dest: /home/week4user/
        owner: week4user
        group: week4user

    - name: Configure sshd, disable root login
      become: true
      lineinfile:
        path: "/etc/ssh/sshd_config"
        regexp: "^PermitRootLogin"
        line: "PermitRootLogin no"
        state: present

    - name: restart sshd in Rocky
      become: true
      systemd:
        daemon-reload: true
        name: sshd
        state: restarted
      when: ansible_distribution in ["Rocky"]
    
    - name: restart sshd in Ubuntu22
      become: true
      systemd:
        name: sshd
        state: restarted
      when: ansible_distribution in ["Ubuntu"]

...
    
```

## Install Podman Playbook with nginx image TASK 2

```bash
vim podman_install.yml
```

```YAML
---

- name: installing podman
  hosts: all
  tasks:
    - name: updating cache in ubuntu
      become: true
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 1000
        name:
          - podman
      when: ansible_distribution in ["Ubuntu"]

    - name: upgrading dnf in rocky
      become: true
      ansible.builtin.dnf:
        name: podman
        state: latest
        update_cache: yes
      when: ansible_distribution in ["Rocky"]

    - name: start podman
      become: true
      systemd:
        daemon-reload: true
        name: podman
        enabled: yes
        state: started

    - name: pull nginx podman image
      podman_image:
        name: docker.io/nginx
        tag: latest

    - name: run nginx image
      containers.podman.podman_container:
        name: nginx_container
        image: docker.io/nginx:latest
        recreate: yes
        state: started
        ports:
          - "8080:8080"


```

