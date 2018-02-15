# Trialing Ansible AWX

The upstream for Ansible Tower , [awx project](https://www.ansible.com/products/awx-project), is in development and worth checking out.  It's pretty painless to get up and running in a testing environment for trial purposes.  You'll need a system running `docker`, `ansible`, and a python dependency, `docker-py`.  There are quite a few ways to set this up, but I'll go over one way, using vagrant and virtualbox to spin up an ubuntu 16.04 machine and walk through installation and some basic usage after it's up and running.  If you already have a linux machine ready, skip the Vagrant lab setup and go straight into Installation.

I've been using chef for a few years now and ansible for about half that time.  I really like how the two compliment each other.  Chef works great for tasks that run constantly, like adding users, and default packages to a newly bootstrapped vm.  I've found that ansible will do just about everything chef can do and found one case that it excels at.  In orchestrating a docker swarm cluster, I tried tackling the challenge first in chef.  When initializing the swarm, the master node creates a unique key that needs applied to each of the other nodes to bring them into the swarm.  With chef, there wasn't an immediate way available for handling this problem without the use of DinamoDB.  Where as in ansible, I was able to save this value to a variable and push it to every node.

So far, I have just been running all of my ansible playbooks from my laptop.  The day has come that I need to centralize all of this config management somewhere the rest of my collogues can see it and join in on the ansible fun I've been having.  I cannot afford to pay for Tower, so I've decided to see what AWX can do for me.

Table of Contents
=================

  * [Requirements](#requirements)
  * [Vagrant](#vagrant)
  * [Installation](#installation)
    * [Ansible](#ansible)
    * [Docker](#docker)
    * [Pip](#pip)
    * [AWX](#awx)

## Requirements

* [Vagrant](https://www.vagrantup.com/docs/installation/)
* [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
* [clone ansible/awx](https://github.com/ansible/awx)
* [install docker](https://docs.docker.com/install/)
* [install ansible 2.4+](http://docs.ansible.com/ansible/latest/intro_installation.html)

## Vagrant

Spin up a couple of virtualbox vms to create a lab environment to test out ansible awx.  One node to host Ansible AWX and one to play the role of a lab host.  If you would like to follow along you can find all of the examples [on github](https://github.com/jahrik/ansible-awx).

Check to see what boxes you have available first and then initialize a new Vagrant file running ubuntu 16.04.

    vagrant box list                                                                                           

    bento/centos-6.8    (virtualbox, 2.3.1)
    bento/centos-7.2    (virtualbox, 2.3.1)
    bento/debian-7.8    (virtualbox, 2.2.1)
    bento/ubuntu-14.04  (virtualbox, 2.3.1)
    bento/ubuntu-16.04  (virtualbox, 2.3.7)
    jhcook/macos-sierra (virtualbox, 10.12.6)
    kennyl/pfsense      (virtualbox, 2.4.0)

    vagrant init bento/ubuntu-16.04

    A `Vagrantfile` has been placed in this directory. You are now
    ready to `vagrant up` your first virtual environment! Please read
    the comments in the Vagrantfile as well as documentation on
    `vagrantup.com` for more information on using Vagrant.

Update this newly created Vagrantfile before bringing up the new machines by adding a few things:
* More RAM
* Bring up a second vm to test against
* Configure hostnames and private IPs
* Run apt-get update

**Vagrantfile**

    # -*- mode: ruby -*-
    # vi: set ft=ruby :

    Vagrant.configure('2') do |config|
      config.vm.box = 'bento/ubuntu-16.04'
      config.vm.provider 'virtualbox' do |vb|
        vb.gui = false
        vb.memory = '4096'
      end
      config.vm.provision 'shell', inline: 'apt-get update'
      {
        'ansible-awx'   => '192.168.33.11',
        'ansible-lab'   => '192.168.33.12'
      }.each do |short_name, ip|
        config.vm.define short_name do |host|
          host.vm.network 'private_network', ip: ip
          host.vm.hostname = "#{short_name}.dev"
        end
      end
    end

Bring up the 2 newly configured vms

    vagrant up
    ...
    ...

Check status when they're finished

    vagrant status
    Current machine states:

    ansible-awx               running (virtualbox)
    ansible-lab               running (virtualbox)


Ssh to a box

    vagrant ssh ansible-awx

You can also open virtualbox and see your newly created vms through the gui and access a console that way.
![virtualbox](https://github.com/jahrik/home_lab/blob/master/ghost/images/virtualbox.png?raw=true)

## Installation

With a fresh installation of ubuntu install any requirements found on the [awx install page for docker-compose](https://github.com/ansible/awx/blob/devel/INSTALL.md#docker-or-docker-compose).  At the time of this writing it includes ansible 2.4+, so we'll need a repo to get that.  I'll ssh into the vagrant box to go over the installation steps.

### Ansible

Install the [latest release via apt ubuntu](http://docs.ansible.com/ansible/latest/intro_installation.html#latest-releases-via-apt-ubuntu)

    sudo apt-get update
    sudo apt-get install software-properties-common
    sudo apt-add-repository ppa:ansible/ansible
    sudo apt-get update
    sudo apt-get install ansible

If I have ansible already installed on my workstation I can handle the above with an ansible playbook to provision the new vms.  First I'll create an inventory.ini file to point ansible at my virtual machines.
NOTE: the default creds for vagrant are used here for simplicity.  These creds are for a lab environment and not expected to be used in production.

**inventory.ini**

    [vagrant]
    ansible-awx ansible_host=192.168.33.11
    ansible-lab ansible_host=192.168.33.12

    [vagrant:vars]
    ansible_connection=ssh
    ansible_ssh_user=vagrant
    ansible_ssh_pass=vagrant


With the inventory.ini file in place I can now point ansible at my new vms

    ansible -i inventory.ini all -m ping

    ansible-awx | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }
    ansible-lab | SUCCESS => {
        "changed": false,
        "ping": "pong"
    }

I would then write a file to handle the above installation of ansible

**install_ansible.yml**

    ---
    - hosts: all
      become: true
      become_method: sudo
      tasks:

      - name: add ansible ppa
        apt_repository:
          repo: 'ppa:ansible/ansible'

      - name: apt-get update && install ansible
        apt:
          update_cache: yes
          name: ansible
          state: present

Finally, run the playbook against the box

    ansible-playbook -i inventory.ini -l ansible-awx install_ansible.yml
    ...
    ...
    ansible-awx                : ok=3    changed=2    unreachable=0    failed=0

Verify the version of ansible running on ansible-awx.  Hopefully, it is 2.4 or higher.

    ansible -i inventory.ini ansible-awx -a 'ansible --version'

    ansible-awx | SUCCESS | rc=0 >>
    ansible 2.4.3.0
      config file = /etc/ansible/ansible.cfg
      configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
      ansible python module location = /usr/lib/python2.7/dist-packages/ansible
      executable location = /usr/bin/ansible
      python version = 2.7.12 (default, Nov 19 2016, 06:48:10) [GCC 5.4.0 20160609]

### Docker

Install the [latest version of docker-engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository) to the ansible-awx vm.

    sudo apt-get update
    sudo apt-get install \
      apt-transport-https \
      ca-certificates \
      curl \
      software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo apt-key fingerprint 0EBFCD88
    sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
      $(lsb_release -cs) \
      stable"
    sudo apt-get update
    sudo apt-get install docker-ce

**install_docker.yml**

    ---
    - hosts: all
      become: true
      tasks:

        - name: add docker engine repo key
          apt_key:
            keyserver: p80.pool.sks-keyservers.net
            id: 58118E89F3A912897C070ADBF76221572C52609D
            state: present

        - name: add docker engine repo
          apt_repository:
            repo: deb https://apt.dockerproject.org/repo ubuntu-xenial main
            state: present
            filename: docker
            update_cache: yes

        - name: install docker-engine
          apt:
            name: docker-engine
            state: present
            update_cache: yes

Install docker-engine on ansible-awx host

    ansible-playbook -i inventory.ini -l ansible-awx install_docker.yml
    ...
    ...
    ansible-awx                : ok=4    changed=3    unreachable=0    failed=0

### Pip

Install pip and docker-py

    sudo apt-get install python-pip

    pip install docker-py

Tack this onto the tail end of docker_install.yml for now and run it again.  It will skip the docker portions because they're already installed.

**tail install_docker.yml**

    - name: install python-pip
      apt:
        name: python-pip
        state: present
        update_cache: yes

    - name: install docker-py
      pip:
        name: docker-py

Run install_docker.yml again to install pip dependencies

    ansible-playbook -i inventory.ini -l ansible-awx install_docker.yml
    ...
    ...
    ansible-awx                : ok=6    changed=2    unreachable=0    failed=0


### AWX

[Clone ansible/awx](https://github.com/ansible/awx) and cd into the new folder.

    git clone https://github.com/ansible/awx.git

    cd awx

Run the install playbook.

    cd installer

    ansible-playbook -i inventory install.yml

And to handle this with ansible instead, here is an example playbook.

**install_awx.yml**

    ---
    - hosts: all
      become: true
      become_method: sudo
      tasks:

      - name: Clone awx from github
        git:
          repo: https://github.com/ansible/awx.git
          dest: /opt/awx/
          version: devel

      - name: Check to see if AWX is running on port 80 and returns a status 200
        ignore_errors: true
        uri:
          url: http://localhost:80
        register: localhost

      - name: output status code
        debug:
          var: localhost.status

      - name: Install and run AWX if it's not already running on port 80
        command: ansible-playbook -i inventory install.yml
        args:
          chdir: /opt/awx/installer/
        when:
          - localhost is defined
          - localhost.status != 200

Run it to install and start up AWX on the ansible-awx host.

    ansible-playbook -i inventory.ini -l ansible-awx install_awx.yml                                
    ...
    ...
    ansible-awx                : ok=4    changed=2    unreachable=0    failed=0

Browse to [192.168.33.11:80](http://192.168.33.11:80) to checkout it out!
![virtualbox](https://github.com/jahrik/home_lab/blob/master/ghost/images/awx_upgrading.png?raw=true)

Use the default creds to log in for the first time.
* admin
* password
![virtualbox](https://github.com/jahrik/home_lab/blob/master/ghost/images/awx_login.png?raw=true)

A brand new installation of AWX :-)
![virtualbox](https://github.com/jahrik/home_lab/blob/master/ghost/images/awx_home.png?raw=true)
