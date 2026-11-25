# Ansible configuration for practical work 11

Run from this directory. Install ansible-core 2.16 or newer and collections:

```sh
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory/bootstrap.ini site.yml --syntax-check
ansible-playbook site.yml --syntax-check
```

Inventory reads the VM host and private-key path from local environment variables. They are not stored in Git. Set DEVOPS_VM_HOST to your verified Host-only address and DEVOPS_SSH_KEY to the local private-key path before running. SSH host keys must be verified before connecting.

The bootstrap inventory uses student with the public key pre-installed using password SSH (the assignment's key bootstrap variant). Run once with sudo password:

```sh
ansible-playbook -i inventory/bootstrap.ini site.yml --ask-become-pass
```

Then use devops by key and non-interactive sudo:

```sh
ansible all -m ping
ansible-playbook site.yml
ansible-playbook verify.yml
```

Roles: base installs official Docker Engine and remote Python dependencies; ssh configures key access, sudo and UFW; nginx manages the reverse proxy; app pulls the reviewed full-SHA image before changing the container and waits for health.

ARM64 adaptation: the VM is aarch64, so the Docker repository architecture is mapped from facts (aarch64 -> arm64, x86_64 -> amd64). The ARM64 image is the reviewed output of practical work 10. No image is built on the VM.

Passwords and private keys must never be committed. `latest` is not used. The VM runs plain HTTP for this assignment; port 5000 is bound only to loopback. `force_handlers = True` prevents notified SSH/Nginx handlers from being lost when a later task fails.
