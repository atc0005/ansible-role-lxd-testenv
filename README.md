# Ansible Role: LXD Test Environment (lxd-testenv)

Ansible role used to generate or teardown a
[LXD](https://linuxcontainers.org/lxd/getting-started-cli/) container test
environment. This role is intended to be used as a base for testing other
roles and playbooks.

[Example playbooks](#example-playbooks) for both use cases are provided.

## Limitations

- Tested on Ubuntu 16.04+ only
  - *support for other distros may be added in a future release*

- This version of the role assumes that the calling playbook provides a host
  list *and* sets the `serial: 1` option
  - each inventory host needs to be processed serially in order to prevent
    race conditions; testing has shown that simultaneous calls to some of the
    modules used (e.g., `known_hosts`) causes race conditions which results in
    data corruption

Future versions of this role are intended to better handle these issues.

## WARNINGS

- Do not use this role on production systems. It lacks sufficient testing to
  be relied upon for anything more than local/isolated testing purposes.

## Requirements

- Ubuntu 16.04 or newer
- [LXD packages](https://linuxcontainers.org/lxd/getting-started-cli/)
  installed
- Equivalent of `sudo lxd init` already run
  - note: the The [`atc0005.lxd-host`
    role](https://github.com/atc0005/ansible-role-lxd-host) role is intended
    to handle this, though as of this writing that role is still incomplete

## Role Variables

- The `state` variable is set outside of the role to act as a control flag for
  either building up a test environment or tearing it down.
  - valid values are `create` or `remove`
- Role variables are set within `default/main.yml`.

| Variable                                     | Default value                        | Purpose                                                                                                                                                                                                                                                                                               |
| -------------------------------------------- | :----------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `lxd_host_ssh_known_hosts_file`              | `$HOME/.ssh/known_hosts`             | Host copy of file. Updated to reflect host keys of all containers.                                                                                                                                                                                                                                    |
| `lxd_host_update_known_hosts_file`           | `true`                               | Controls whether the `lxd_host_ssh_known_hosts_file` file on the host is updated with host keys from all containers.                                                                                                                                                                                  |
| `lxd_host_update_etc_hosts_file`             | `true`                               | Controls whether the `/etc/hosts` file on the LXD container host is updated to include new container name/IP pairs                                                                                                                                                                                    |
| `lxd_host_ssh_private_key_for_containers`    | `$HOME/.ssh/id_ed25519`              | Generated if it does not exist. See notes for public key.                                                                                                                                                                                                                                             |
| `lxd_host_ssh_public_key_for_containers`     | `$HOME/.ssh/id_ed25519.pub`          | Inserted into `root` and `ansible` user profiles within each container.                                                                                                                                                                                                                               |
| `lxd_host_remove_container_settings`         | `false`                              | Controls whether this role removes `lxd_host_ssh_known_hosts_file` and `/etc/hosts` entries as part of teardown tasks. Usually, but not required to be performed along with container removal. Defaults to `false`                                                                                    |
| `lxd_host_remove_containers`                 | `false`                              | Controls whether containers are removed as part of teardown process. Defaults to `false`.                                                                                                                                                                                                             |
| `lxd_containers_update_hosts_file`           | `false`                              | Controls whether `/etc/hosts` within containers are updated to include entries for *all* containers generated by this role. Since the LXD-provided dnsmasq instance bound to the bridge already handles name resolution, this option is not really needed and may be removed in a future role update. |
| `lxd_containers_image_server`                | <https://images.linuxcontainers.org> | Server listed in the Ansible module examples. Used to fetch LXD container images for generating fresh containers.                                                                                                                                                                                     |
| `lxd_containers_create`                      | `true`                               | Controls whether containers are generated when this role is run. **TODO**: Validation is needed to assert that this is not set at the same time as the flag for removing containers.                                                                                                                  |
| `lxd_containers_bootstrap`                   | `true`                               | Controls whether Python, sudo and other packages are installed as part of preparing new containers for management by Ansible.                                                                                                                                                                         |
| `lxd_containers_configure`                   | `true`                               | Create service account, groups, enable SSH at boot, etc                                                                                                                                                                                                                                               |
| `lxd_containers_packages_required`           | `openssh-server`, `sudo`             | Packages installed as part of the *bootstrap* process for new containers                                                                                                                                                                                                                              |
| `lxd_containers_packages_extra`              | `nano`                               | Ensure that a sensible (j/k) editor is available in newly created containers                                                                                                                                                                                                                          |
| `lxd_containers_profiles`                    | `default`                            | LXD profiles applied to all new containers by default.                                                                                                                                                                                                                                                |
| `docker_main_config_file`                    | `/etc/docker/daemon.json`            | Path to Docker "daemon" configuration file. Role-provided template for this file configures the active storage driver used within new containers intended to test Docker workflows.                                                                                                                   |
| `lxd_containers_docker_config_file_template` | `docker-daemon-config.json.j2`       | Template file included with role used for setting Docker daemon storage driver.                                                                                                                                                                                                                       |
| `lxd_containers_docker_storage_driver`       | `vfs`                                | Docker storage drivers: `overlay2` is incompatible with LXD LTS (2.0, 3.0), `overlay` is incompatible with ZFS backed LXD storage. `vfs` is slow, but compatible with both. Override this from a playbook (or other higher level Ansible variable source) if not using ZFS for better performance.    |
| `lxd_containers_sudoers_include_file`        | `/etc/sudoers.d/ansible`             | Location of sudoers include file used to grant service account full sudo privileges within new containers                                                                                                                                                                                             |
| `lxd_containers_service_account`             | `ansible`                            | Service account created within containers that is granted full sudo privileges. Often used by playbooks as a dedicated remote management account.                                                                                                                                                     |
| `lxd_containers_service_group`               | `ansible`                            | Service group that goes with the service account already noted here.                                                                                                                                                                                                                                  |
| `state`                                      | `create`                             | Flag with valid values of `create` or `remove`. Used to control whether a test environment will be created or removed/destroyed.                                                                                                                                                                      |
| `default_python_install_command`             | *see example below*                  | Crude one-liner shell command to use appropriate package manager to install Python for Ansible use via a `raw` command.                                                                                                                                                                               |
| `python_path`                                | `/usr/bin/python`                    | Location of Python interpreter used by Ansible on remote hosts (containers in our case).                                                                                                                                                                                                              |

Default shell command for the `default_python_install_command` variable:

```shell
if [[ $? -ne 0 ]]; then
  if [[ -f /usr/bin/apt-get ]]; then
    apt-get install -y python;
  fi
  if [[ -f /usr/bin/yum ]]; then
    yum install -y python;
  fi
fi
```

If not set elsewhere and if Python 2.x is not found on the system as
`/usr/bin/python`, the above command is used in an attempt to install Python 2
for further Ansible management steps.

## Dependencies

An existing installation of [LXD
packages](https://linuxcontainers.org/lxd/getting-started-cli/) and initial
configuration via `sudo lxd init`. The [`atc0005.lxd-host`
role](https://github.com/atc0005/ansible-role-lxd-host) is intended to serve
that purpose in the future once it is ready.

See [Requirements](#requirements) section for additional items.

## Example Playbooks

### Setup test environment

**WARNING:** Review known [Limitations](#limitations) before using this role.

```yaml

- name: Configure base LXD host
  hosts: localhost
  connection: local

  # Should be fine to gather facts for just the future LXD host
  gather_facts: yes

  tasks:

    # Stub role for now. Later work will have it install required packages
    # needed in order to configure host system for running LXD containers
    - name: Apply lxd-host role to install LXD packages, perform initial setup
      import_role:
        name: atc0005.lxd-host

- name: Setup LXD test environment
  hosts: all
  connection: local

  # This is needed due to potential for multiple access attempts via modules
  # that are not intended/designed for it. Without this setting occasional
  # race conditions occur which results in data corruption.
  serial: 1

  # Rely on specific tasks to call setup module as needed
  gather_facts: no

  tasks:

    # Default role settings handle creation; we have to explicitly enable
    # container/settings removal tasks
    - name: Apply lxd-testenv role to create test environment
      import_role:
        name: atc0005.lxd-testenv
      vars:
        # Valid values: "create" and "remove"
        state: "create"
```

### Teardown/Remove test environment

**WARNING:** Review known [Limitations](#limitations) before using this role.

```yaml

- name: Tear down our LXD test environment
  hosts: all
  connection: local
  gather_facts: no

  # This is needed due to potential for multiple access attempts via modules
  # that are not intended/designed for it. Without this setting occasional
  # race conditions occur which results in data corruption.
  serial: 1

  tasks:

    - name: Apply lxd-testenv role
      import_role:
        name: atc0005.lxd-testenv
      vars:
        # Valid values: "create" and "remove"
        state: "remove"
        lxd_host_remove_container_settings: true
        lxd_host_remove_containers: true

```

## Installing the role

### Create `requirements.yml` file

Note: This role is not yet availalble via Ansible Galaxy, so for now you will
need to install it via a `requirements.yml` file.

#### Specific stable release version

```yaml
---

- name: "atc0005.lxd-testenv"
  src: https://github.com/atc0005/ansible-role-lxd-testenv.git"
  version: "v1.0"

...

```

#### Latest from the `master` branch

Example `requirements.yml` file that uses the latest from the `master` branch:

```yaml
---

- name: "atc0005.lxd-testenv"
  src: https://github.com/atc0005/ansible-role-lxd-testenv.git"
  version: "master"

...

```

### Use `requirements.yml` file to install role

- Because `ansible-galaxy` does not yet natively support updating existing
  roles, the example installation instructions provided below include the
  `--force` parameter to *force* pulling in the latest updates as directed by
  the `requirements.yml` file.

#### Option 1: Within your home directory

- `ansible-galaxy install -r requirements.yml --force`

#### Option 2: Within a `roles` directory in the same location as your playbooks

- `ansible-galaxy install -r requirements.yml -p roles --force`

#### Option 3: System-wide for all users

- `sudo ansible-galaxy install -r requirements.yml --force`

## License

MIT

## Author Information

This role was created in 2019 by Adam Chalkley as part of building an
on-demand development environment toolkit managed by Ansible.

## References

- <https://github.com/atc0005/ansible-playbooks>
- <https://github.com/atc0005/ansible-role-lxd-host>
- <https://github.com/atc0005/ansible-role-lxd-testenv>
- <https://github.com/atc0005/ansible-role-docker-host>

- <https://linuxcontainers.org/lxd/getting-started-cli/>

- <https://docs.ansible.com/ansible/latest/modules/lxd_container_module.html>
- <https://docs.ansible.com/ansible/latest/modules/lxd_profile_module.html>
