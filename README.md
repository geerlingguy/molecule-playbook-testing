# Ansible 101 - Molecule playbook testing

> _This example is derived from Chapter 12 of [Ansible for DevOps](https://www.ansiblefordevops.com), version 1.23_

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

Then run:

    molecule converge

Uh oh... something failed. Log in and check it out:

```
$ molecule login
[root@instance /]# systemctl status httpd
Failed to get D-Bus connection: Operation not permitted
```

Yikes, I can't take this entire episode to explain the details of what's going on, but at a basic level, Molecule's default configuration sets up a container and runs the command:

    bash -c "while true; do sleep 10000; done"

(use `docker ps` to confirm that.)

Unfortunately, this means the process that's running in this container is _not_ systemd's init system, meaning you can't use systemd to manage services like Apache.

In normal container usage, this is perfectly fine—running an init system in a container is kind of an anti-best-practice.

But in our case, we're using containers for testing. And lucky for you, I maintain a large set of Ansible testing containers that have systemd built in and ready to go! So for these tests, since we want to make sure Ansible can manage the Apache service correctly, we need to change a few things in Molecule's `molecule.yml` configuration.

Go ahead and exit and destroy the environment.

```
[root@instance /]# exit
$ molecule destroy
```

Now edit the `platforms` in `molecule.yml`:

```yaml
platforms:
  - name: instance
    image: "geerlingguy/docker-${MOLECULE_DISTRO:-centos7}-ansible:latest"
    command: ""
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    pre_build_image: true
```

Now try running it again:

    molecule test

Yay, it works!

Now, since we added an environment variable, `MOLECULE_DISTRO`, we can substitute whatever distro we want there, assuming I maintain a Docker image for that distro:

    MOLECULE_DISTRO=debian10 molecule test

Yay, still works!

## Verify with Molecule

Now add to `verify.yml`:

```yaml
---
- name: Verify
  hosts: all

  tasks:
  - name: Verify Apache is serving web requests.
    uri:
      url: http://localhost/
      status_code: 200
```

You can just run the verify playbook with `molecule verify`.

## Add lint configuration

Assuming you have `yamllint` and `ansible-lint` installed, you can configure Molecule to run them in the `lint` build stage:

```yaml
lint: |
  set -e
  yamllint .
  ansible-lint
```

Then run `molecule lint`. It doesn't even need to build an environment to do the linting.

## GitHub Actions Testing

Go ahead and slap this repo up on GitHub. Do it!

Now add a `.github/workflows` folder.

Add a `ci.yml` workflow file.

Here's the whole workflow we're targeting:

```yaml
---
name: CI
'on':
  pull_request:
  push:
    branches:
      - master

jobs:

  test:
    name: Molecule
    runs-on: ubuntu-latest
    strategy:
      matrix:
        distro:
          - centos8
          - debian10

    steps:
      - name: Check out the codebase.
        uses: actions/checkout@v2

      - name: Set up Python 3.
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'

      - name: Install test dependencies.
        run: pip3 install molecule docker yamllint ansible-lint

      - name: Run Molecule tests.
        run: molecule test
        env:
          PY_COLORS: '1'
          ANSIBLE_FORCE_COLOR: '1'
          MOLECULE_DISTRO: ${{ matrix.distro }}
```
