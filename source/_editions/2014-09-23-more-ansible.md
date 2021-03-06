---
title: Ansible Facts & Vault
topics: [Ansible Facts, Ansible Vault]
description: See how to use Ansible Facts to gather system information and Anisble Vault to encrypt data.
---

<a name="ansible_facts" id="ansible_facts"></a>
## Facts

Before running any Tasks, Ansible will gather information about the system it's provisioning. These are called Facts, and include a wide array of system information such as the number of CPU cores, available ipv4 and ipv6 networks, mounted disks, Linux distribution and more.

Facts are often useful in Tasks or Tempalte configurations. For example Nginx is commonly set to use as any worker processors as there are CPU cores. Knowing this, you may choose to set your template of the `nginx.conf` file like so:

```nginx
user www-data www-data;
worker_processes {% verbatim %}{{ ansible_processor_cores }}{% endverbatim %};
pid /var/run/nginx.pid;

# And other configurations...
```

Or if you have a server with multiple CPU's, you can use:

```nginx
user www-data www-data;
worker_processes {% verbatim %}{{ ansible_processor_cores * ansible_processor_count }}{% endverbatim %};
pid /var/run/nginx.pid;

# And other configurations...
```

Ansible facts all start with `anisble_` and are globally available for use any place variables can be used: Variable files, Tasks, and Templates.

### Example: NodeJS

For Ubuntu, we can get the latest stable NodeJS and NPM from NodeSource, which has teamed up with Chris Lea. Chris ran the Ubuntu repository `ppa:chris-lea/node.js`, but now provides NodeJS via NodeSource packages. To that end, they have provided a shells script which installs the latest stable NodeJS and NPM on Debian/Ubuntu systems.

This shellscript is found at [https://deb.nodesource.com/setup](https://deb.nodesource.com/setup). We can take a look at this and convert it to the following tasks from a NodeJS Role:

```yaml
---
- name: Ensure Ubuntu Distro is Supported
  get_url: 
    url='https://deb.nodesource.com/node/dists/{% verbatim %}{{ ansible_distribution_release }}{% endverbatim %}/Release' 
    dest=/dev/null
  register: distrosupported

- name: Remove Old Chris Lea PPA
  apt_repository: 
    repo='ppa:chris-lea/node.js' 
    state=absent
  when: distrosupported|success

- name: Remove Old Chris Lea Sources
  file: 
    path='/etc/apt/sources.list.d/chris-lea-node_js-{% verbatim %}{{ ansible_distribution_release }}{% endverbatim %}.list' 
    state=absent
  when: distrosupported|success

- name: Add Nodesource Keys
  apt_key: 
    url=https://deb.nodesource.com/gpgkey/nodesource.gpg.key 
    state=present

- name: Add Nodesource Apt Sources List Deb
  apt_repository: 
    repo='deb https://deb.nodesource.com/node {% verbatim %}{{ ansible_distribution_release }}{% endverbatim %} main' 
    state=present
  when: distrosupported|success

- name: Add Nodesource Apt Sources List Deb Src
  apt_repository: 
    repo='deb-src https://deb.nodesource.com/node {% verbatim %}{{ ansible_distribution_release }}{% endverbatim %} main' 
    state=present
  when: distrosupported|success

- name: Install NodeJS
  apt: pkg=nodejs state=latest update_cache=true
  when: distrosupported|success
```

There's a few tricks happening there. These mirror the bash script provided by Node Source.

First we create the `Ensure Ubuntu Distro is Supported` task, which uses the `ansible_distribution_release` Fact. This gives us the Ubuntu release, such as Precise or Trusty. If the resulting URL exists, then we know our Ubuntu distribution is supported and can continue. We register `distrosupported` so we can test if this step was successfully on subsequent tasks.

Then we run a series of tasks to remove NodeJS repositories in case the system already has `ppa:chris-lea/node.js` added. These only run when if the distribution is supported via `when: distrosupported|success`. Note that most of these continue to use the `ansible_distribution_release` Fact.

Finally we get the debian source packages and install NodeJS after updating the repository cache. This will install the latest stable of NodeJS and NPM. We know it will get the latest version available by using `state=latest` when installing the `nodejs` package.

---

<a name="ansible_vault" id="ansible_vault"></a>
## Vault

We often need to store sensitive data in our Ansible templates, Files or Variable files; It unfortunately cannot always be avoided. Ansible has a solution for this called Ansible Vault.

Vault allows you to encrypt any Yaml file, which typically boil down to our Variable files. Vault will not encrypt Files and Templates. 

When creating an encrypted file, you'll be asked a password which you must use to edit the file later and when calling the Roles or Playbooks.

For example we can create a new Variable file:

```bash
ansible-vault create vars/main.yml
Vault Password:
```

After entering in the encryption password, the file will be opened in your default editor, usually Vim. 

The editor used is defined by the `EDITOR` environmental variable. The default is usually Vim. If you are not a Vim user, you can change it quickly by setting the environmental variabls:

```bash
export EDITOR=nano
ansible-vault edit vars/main.yml
```

T> The editor can be set in the users profile/bash configuration, usually found at `~/.profile`, `~/.bashrc`, `~/.zshrc` or similar, depending on the shell and Linux distribution used.

Ansible Vault itself is fairly self-explanatory. Here are the commands you can use:

```bash
$ ansible-vault -h
Usage: ansible-vault [create|decrypt|edit|encrypt|rekey] \
      [--help] [options] file_name

Options:
  -h, --help  show this help message and exit
```

For the most part, we'll use `ansible-vault create|edit /path/to/file.yml`. Here, however, are all of the available commands:

* **create** - Create a new file and encrypt it
* **decrypt** - Create a plaintext file from an encrypted file
* **edit** - Edit an already-existing encrypted file
* **encrypt** - Encrypt an existing plain-text file
* **rekey** - Set a new password on a encrypted file

### Example: Users

I use Vault when creating new users. In a User Role, you can set a Variable file with users' passwords and a public key to add to the users' authorized_keys file (thus giving you SSH access).

T> Public SSH keys are technically safe for the general public to see - all someone can do with them is allow you access to their own servers. Public keys are intentionally useless for gaining access to a sytem without the paired private key, which we are not putting into this Role.

Here's an example variable file which can be created and encrypt with Vault. While editing it, it's of course in plain-text:

```yaml
admin_password: $6$lpQ1DqjZQ25gq9YW$mHZAmGhFpPVVv0JCYUFaDovu8u5EqvQi.Ih
deploy_password: $6$edOqVumZrYW9$d5zj1Ok/G80DrnckixhkQDpXl0fACDfNx2EHnC
common_public_key: ssh-rsa ALongSSHPublicKeyHere
```

Note that the passwords for the users are also hashed. You can read Ansible's documentation on [generating encrypted passwords](http://docs.ansible.com/faq.html#how-do-i-generate-crypted-passwords-for-the-user-module), which the User module requires to set a user password. As a quick primer, it looks like this:

```bash
# The whois package makes the mkpasswd 
# command available on Ubuntu
$ sudo apt-get install -y whois

# Create a password hash
$ mkpasswd --method=SHA-512
Password:
```

This will generate a hashed password for you to use with the `user` module.

Once you have set the user passwords and added the public key into the Variables file, we can make a Task to use these encrypted variables:

```yaml
---
- name: Create Admin User
  user: 
    name=admin 
    password={% verbatim %}{{ admin_password }}{% endverbatim %} 
    groups=sudo 
    append=yes 
    shell=/bin/bash

- name: Add Admin Authorized Key
  authorized_key: 
    user=admin
    key="{% verbatim %}{{ common_public_key }}{% endverbatim %}"
    state=present

- name: Create Deploy User
  user: 
    name=deploy 
    password={% verbatim %}{{ deploy_password }}{% endverbatim %} 
    groups=www-data 
    append=yes 
    shell=/bin/bash

- name: Add Deployer Authorized Key
  authorized_key: 
    user=deploy
    key="{% verbatim %}{{ common_public_key }}{% endverbatim %}"
    state=present
```

These Tasks use the `user` module to create new users, passing in the passwords set in the Variable file.

It also uses the `authorized_key` module to add the SSH pulic key as an authorized SSH key in the server for each user.

Variables are used like usual within the Tasks file. However, in order to run this Role, we'll need to tell Ansible to ask for the Vault password so it can unencrypt the variables.

Let's setup a `provision.yml` Playbook file to call our `user` Role:

```yaml
---
- hosts: all
  sudo: yes
  roles:
    - user
```

To run this Playbook, we need to tell Ansible to ask for the Vault password, as we're running a Role which contains an encrypted file:

```bash
ansible-playbook --ask-vault-pass provision.yml
```
