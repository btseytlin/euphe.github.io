---
published: false
---
## How to run Ansible on Windows (or anywhere else) with Docker

If you have ever been to a situation, where you needed to run Ansible playbooks from a windows control machine, you have already found out **Ansible isn't built to do that**.

But you are on Windows, you need to do your work. 
So here's a simple solution: **make a Docker container**.

Isn't it the solution to everything nowadays?

Here's what you should do.

1. Take this Dockerfile: [Github Gist](https://gist.github.com/Euphe/5cabd9ceb211d97230e9a8bf757dc47b "Ansible on windows Dockerfile")

It's a slightly modified `Dockerfile` from [this post](https://medium.com/@tech_phil/running-ansible-inside-docker-550d3bb2bdff).
It installs a barebones Linux (alpine), installs ansible and it's dependencies, installs bash (alpine doesn't include bash by default).
The original `Dockerfile` is limited to running playbooks. But Ansible has **so much more** (e.g. ansible-vault, ad-hoc commands, Ansible Galaxy). 
My approach will allow you to use Ansible just like you would use it on Linux.

2. Take this [docker-compose.yml](https://gist.github.com/Euphe/51f9011b4dd1fef304493728e47c5753) for convinience

3. Arrange your directory structure this way:
```
- project/
  - Dockerfile
  - docker-compose.yml
  - ansible.cfg
  - playbook.yml
  - roles/
```

4. `docker-compose up -d`
```
$ docker-compose up -d
Creating project_ansible_1 ...
Creating project_ansible_1 ... done
```
What happens next: the image is built, the container is run in detached mode.

5. Attach to it for the bash session

```
$ docker exec -it prod_ansible_1 /bin/bash
bash-4.3# ansible --version
ansible 2.2.2.0
  config file = /ansible/exploitation/ansible.cfg
  configured module search path = Default w/o overrides
bash-4.3#
```
Now you can run whatever Ansible commands you want.

6. Exec any command without attaching
```
$ docker exec -it prod_ansible_1 ansible --version
ansible 2.2.2.0
  config file = /ansible/exploitation/ansible.cfg
  configured module search path = Default w/o overrides
```

### Done!

Now you can take advantage of Ansible as if you were on Linux.
This approach is not much different from the common "run a Linux VM with Vagrant".
But it's simple, and only requires you to copy paste two files and run a command. 


