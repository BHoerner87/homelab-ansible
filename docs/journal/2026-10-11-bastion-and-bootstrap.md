First sitting of work learning ansible in my home lab in order to better manage it, and begin learning ansible principles.

I have two main PCs as servers in my environment, one which hosts baremetal software services, and the other which runs Proxmox. In the Proxmox server, I created a new VM to exist as a bastion host for managing the other servers.

I turned this new bastion VM into an Ansible control node with its work backed up to GitHub. The idea there is to track progress as a kind of portfolio effort, and to ensure that if something should ever happen to that machine, setting this control node back up should not be difficult.

I made two passphrase-protected keys (one for GitHub, and one for the fleet of other machines), and used ssh-agent through the keychain to keep them unlocked. I also created two side-by-side repos in GitHub: a public homelab-ansible repo for the code people will be allowed to see, and a private homelab-inventory for hosts and secrets.

I wrote an inventory that groups hosts by type and exposure, pre-recorded host keys, and wrote a bootstrap playbook that gives each host an ansible user with the fleet key and passwordless sudo. This is working in the Minecraft LXC, which required slightly different attention because it is a container. Along the way, I also corrected an oversight on the vm1-web VM which runs a number of Docker containers managed by Portainer, and which had an admin account but which password I never originally recorded. I set a new, unique, strong password for this account and recorded it in my password manager to adhere to best practices and avoid logging in with root.

The next step will be to bootstrap remaining hosts, confirm ```ansible all -m ping``` is all green, and then, the ```ssh_hardening``` role that turns off password login.

## Terms learned
**Control node / managed node**: the machine running Ansible (the bastion host) / the machines it manages
**Inventory**: The file listing hosts and their groups (hosts.yml)
**Module**: one unit of work, like apt, user, or copy. Each task calls one of these.
**Task / play / playbook**: one module call / a set of tasks aimed at a set of hosts / a YAML file containing plays
**Collection**: a bundle of modules under namespace.collection (e.g., ansible.posix)
**FQCN**: fully-qualified collection name, e.g. namespace.collection.module
**Galaxy / ansible-galaxy**: Ansible's public registry / the command for installing from it and scaffolding roles
**become**: Ansible's term for privilege escalation (sudo)
**Idempotent**: running it again changes nothing if things are already correct (**ok** vs **changed**)
**sshpass**: feeds a password to SSH non-interactively; only needed for --ask-pass
