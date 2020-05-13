# Molecule playbook testing

This playbook installs Apache and should work on the latest versions of Red Hat-based and Debian-based OSes, at a minimum:

  - CentOS 8
  - Debian 10

Create a new molecule scenario:

    molecule init scenario

Edit the `converge.yml` playbook:

```yaml
---
- name: Converge
  hosts: all

  tasks:
    - name: Update apt cache (on Debian).
      apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == 'Debian'

- import_playbook: ../../main.yml
```

TODO.
