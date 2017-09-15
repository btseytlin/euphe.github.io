---
title: Proper workflow to deploy something with Ansible and Docker.
date: 2016-02-09 09:27:00 +03:00
categories:
- devops
- dont-make-my-mistake
tags:
- ansible
- docker
- gitlab
excerpt: Don't make my mistake of believing Ansible will do everything for you.
comments: true
---

### Pre-note

To understand the problem of this article a __basic__ understanding of Docker, Ansible and the deployment process is required.

# Preambula

I have just gotten my job 2 days before when I was tasked with setting up [Gitlab](http://gitlab.com/) in a [Docker](http://docker.com/) container, and create an [Ansible](http://ansible.com) playbook to automate the deployment process.

### What are Docker and Ansible

I spent half a day reading through docs to learn about the tools I need to use.
Here's the __tl;dr__ on Docker and Ansible.

`docker` runs an application inside a __container__, a tiny virtual machine that's supposed to run a single process. `docker-compose` allows to run multiple containers, creating a virtual network of them. For example you could have one container for a webserver, one container for a database, one container for your app.  
This is a great article to get the gist of docker: (Docker explained simply)[http://elliot.land/post/docker-explained-simply].
Generally docker containers are __super cool__. They allow you to completely control the environment of any app. Containers are very easy to destroy: if you botched the config you can just destroy the container and make a new one with proper configs and be sure that your host system remains unaffected.

Ansible is a tool to remotely control and configure servers. The basic example is that you can run a command across a thousand servers and see the output. A thousand servers, however, is not a daily work situation for most of us. Ansible is much more with __playbooks__, which are basically configuration scripts, written in __YAML__. Ansible playbooks are readable even without knowing a thing about Ansible. I found them to be much more of a "programmer" way to do system administration: you create a playbook that does the same thing every time you run it, you test it and make sure it works, you add variables to it so you can reuse the playbook on other servers, you add it to your version control so your teammates can run it. Much like developing a program! 
The official Ansible introduction is honestly the best to get hang of it: [How Ansible works](https://www.ansible.com/how-ansible-works).
But really, all of Ansible docs are great. You can read them in a day and know most things you need to know.

### Their powers, combined

Here's the big deal: you can combine Docker and Ansible. 
There are three ways to achieve it:
1. Use Ansible to run the `docker` command on target host.
Example of using Ansible to run the command taken from [Gitlab docs](https://docs.gitlab.com/omnibus/docker/README.html#run-the-image):
```yaml
---
- hosts: [gitlab]
  become: true

  tasks:
    - name: start gitlab docker container
      command: >
        sudo docker run --detach
        --hostname gitlab.example.com
        --publish 443:443 --publish 80:80 --publish 22:22
        --name gitlab
        --restart always
        --volume /srv/gitlab/config:/etc/gitlab
        --volume /srv/gitlab/logs:/var/log/gitlab
        --volume /srv/gitlab/data:/var/opt/gitlab
        gitlab/gitlab-ce:latest

``` 
Put it in 'gitlab.yml', add a 'gitlab' group to Ansible inventory, run 'ansible-playbook gitlab' and __bam__, your host has Gitlab fully setup.
Bonus: You can add [variables]() to change the hostname, ports, and gitlab image from a config file. Yay, reusability!

2. Create a local `docker-composes.yml` file, deploy it to target host, run `docker-compose` command on host.
Example:
```yaml
---
- hosts: [gitlab]
  become: true
  tasks:
    - name: copy gitlab docker_compose
      copy:
        src: ./docker_composes/gitlab.yml
        dest: ~/gitlab.yml

    - name: start gitlab docker containers
      command: docker-compose -f ~/gitlab.yml up -d
```
Bonus: You can make `docker-composes/gilab.yml` a `.j2` [template](http://docs.ansible.com/ansible/latest/playbooks_templating.html) to get the reusability benefit of varibles.

3. Use Ansible's [`docker_container`](http://docs.ansible.com/ansible/docker_container_module.html).
You get the benefits of `docker-composes` with the readability of keeping everything in the same file.
You can even declare multiple containers and link them to eachother, like you would in a `docker-composes` file.
And you can use the variables right here.
Example:
```yaml
---
- hosts: [gitlab]
  become: true
  tasks:
    - name: Install gitlab container 
      docker_container:
        name: gitlab
        detach: true
        image: 'gitlab/gitlab-ce:latest'
        state: started
        restart_policy: always
        hostname: "{{gitlab_hostname}}"
        env:
          GITLAB_OMNIBUS_CONFIG: "{{GITLAB_OMNIBUS_CONFIG}}"
        ports:
          - '{{gitlab_http_port}}:80'
          - '{{gitlab_https_port}}:443'
          - '{{gitlab_ssh_port}}:22'
        volumes:
          - "{{host_gitlab_directory_path}}/config:/etc/gitlab"
          - "{{host_gitlab_directory_path}}/logs:/var/log/gitlab"
          - "{{host_gitlab_directory_path}}/data:/var/opt/gitlab"

``` 
You guessed it, this is my favourite version.

# My mistake

I really liked Ansible. It seemed like system administration will never be a pain again. It seemed like you don't need to do anything by hand. Good bye one-off solutions, now you can write a playbook ( or just [download it](https://galaxy.ansible.com/) ) and get a configurable, reusable, general solution. 