# Ansible for user management

Ansible is a decent option for user management is because it is idempotent. This means, Ansible will only make a change if the change is needed to get to the desired state. So you can run the same Ansible Playbook multiple times, and if the Ansible tasks are written properly Ansible will only make a change the first time.

Keep in mind, this solution is not “enterprise” grade. But it certainly works for certain scenarios, like in a lab environment. 

This Ansible role is tested on:

- Ubuntu
- Red Hat Enterprise Linux (RHEL) 7.x, 6.5, 5.9
- CentOS 7.x, 6.5, 5.9


## What this user management Ansible role does

This role will manage users in the user list configuration file (list is in the file vars/lab_users.secret in the example below).
It can add users, change passwords, lock/unlock user accounts, manage sudo access (per user), add ssh key(s) for ssh key based authentication.
This is done on a per “group” basis (using Ansible group variables), as set in the configuration file. The group comes from the Ansible group as set for a server in the inventory file.

**Note:** *Deleting users is not done on purpose.*

The great thing about Ansible Playbooks, if you don’t like something it’s easy enough to modify.


## Install the Ansible user management role

First let’s install the Ansible role to manage users into your Ansible control server:

```bash
$ su - ansible
$ cd ~/ansible
$ git clone https://github.com/cubit9/ansible-role-users.git roles/create-users
```

Or, you can use Ansible Galaxy to install this role:

```bash
$ su - ansible
$ ansible-galaxy install ryandaniels.create_users
```


## Setup Ansible Vault

If you don’t already have [Ansible Vault](https://docs.ansible.com/ansible/latest/vault.html) configured, you will need to set it up now. Vault uses AES encryption to store your sensitive information. And we need Ansible Vault to encrypt our user configuration file since we don’t want to be exposing even a hashed password into source control.

First, (if using git), make sure to update your .gitignore file so your Vault password isn’t saved to your source control. And also add the secret file we will be creating later:

```bash
$ vi .gitignore
.vaultpass
secret
*.secret
```

Next, create a password for your Ansible Vault and save it in your Password Manager (like [KeePassXC](https://keepassxc.org/)). Then create the `.vaultpass` file, add the Vault password, and fix the permissions:

```bash
$ vi .vaultpass
#Enter password here

$ chmod 600 .vaultpass
```

Also you need to update the Ansible config file to reference where the Vault file is located:

```bash
$ vi ansible.cfg
[defaults]
vault_password_file = ./.vaultpass
```


## Create Ansible Playbook

Next, create the Playbook file:

```bash
$ vi create-users.yml
---
- hosts: '{{inventory}}'
  vars_files:
    - vars/lab_users.secret
  become: yes
  roles:
  - create-users
```


## Create configuration for user management

Next, add your users into a configuration file. The below is only an example, don’t use it in your user management configuration file. Also, make sure the filename matches a .gitignore entry. In this case it will match “*.secret”.

Use the special Ansible command to create the encrypted Ansible Vault file:

```bash
$ mkdir -p vars
$ ansible-vault create vars/lab_users.secret
---
users:
  - username: alice
    password: $6$/y5RGZnFaD3f$96xVdOAnldEtSxivDY02h.DwPTrJgGQl8/MTRRrFAwKTYbFymeKH/1Rxd3k.RQfpgebM6amLK3xAaycybdc.60
    update_password: on_create
    comment: Test User 100
    shell: /bin/bash
    ssh_key: |
      ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx8crAHG/a9QBD4zO0ZHIjdRXy+ySKviXVCMIJ3/NMIAAzDyIsPKToUJmIApHHHF1/hBllqzBSkPEMwgFbXjyqTeVPHF8V0iq41n0kgbulJG alice@laptop
      ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx8crAHG/a9QBD4zO0ZHIjdRXy+ySKviXVCMIxxxxxxxxxxxxxxxxxxJmIApHHHF1/hBllqzBSkPEMwgFbXjyqTeVPHF8V0iq41n0kgbulJG alice@server1
    exclusive_ssh_key: yes
    use_sudo: no
    use_sudo_nopass: no
    user_state: present
    servers:
      - webserver
      - database
      - monitoring

  - username: bob
    password: $6$XEnyI5UYSw$Rlc6tXtECtqdJ3uFitrbBlec1/8Fx2obfgFST419ntJqaX8sfPQ9xR7vj7dGhQsfX8zcSX3tumzR7/vwlIH6p/
    ssh_key: AAAAB3NzaC1yc2EAAAADAQABAAACAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx8crAHG/a9QBD4zO0ZHIjdRXy+ySKviXVCMIxxxxxxxxxxxxxxxxxxJmIApHHHF1/hBllqbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbulJG bob@laptop
    use_sudo: no
    user_state: lock
    servers:
      - webserver
      - database
```

In the above example, there are two users. Users: alice and bob. If you are familiar with user management on Linux, then most of these settings are self explanatory. For example, both Alice and Bob have a hashed password and ssh keys. Alice actually has two ssh keys configured. But Bob’s account is locked so he can’t log in.

The interesting part is the list of “servers” in the configuration. These are groups defined in your Ansible hosts file. So you can give certain users access to a specific group of servers, depending on the user’s role.

Important note: Be careful with the update_password setting. When set to always, the user’s password will be changed to what is defined in password. This might not be what they wanted if they’ve manually changed their password so it’s usually safer to use on_create.

For details about all the different settings, see the [README](https://github.com/cubit9/ansible-role-users/blob/master/README.md#user-settings) on GitHub.

Going forward, to update the file, instead of the “create” command line option, use “edit”:

```bash
$ ansible-vault edit vars/lab_users.secret
```

And don’t forget to save this into your source control (git).


## Run the Ansible Playbook

```bash
$ ansible-playbook create-users.yml --extra-vars "inventory=all-dev" -i hosts-dev
```

Marvel at the output generated by Ansible. All these users are created, updated, or locked, but only if something needed to change since Ansible is idempotent.


## Bonus: Add this in Jenkins

If you also use Jenkins for your CI/CD pipeline, you can add this Ansible Playbook into Jenkins.

You just need to setup Jenkins to use Ansible and Ansible Vault. And there is a great Jenkins Plugin to help. Just follow this guide to use [Ansible Vault with Jenkins](https://ryandaniels.ca/blog/ansible-vault-jenkins/). Keep in mind that updating the user configuration file is only possible from the command line, since it’s encrypted using Ansible Vault.