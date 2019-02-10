# Ansible Role: LXD Test Environment (lxd-testenv)

Ansible role used to generate or teardown a
[LXD](https://linuxcontainers.org/lxd/getting-started-cli/) container test
environment. This role is intended to be used as a base for testing other
roles and playbooks.

[Example playbooks](#example-playbooks) for both use cases are provided.

## Limitations

- Tested on Ubuntu 16.04+ with LXD 2.0 and 3.0 LTS series
  - *support for other distros and LXD versions may be added in a future
    release*

- This version of the role assumes that the calling playbook specifies
  `localhost` only as the sole entry in the `hosts` list. Attempting to use
  `all` as a value for that parameter results in playbook/role failure.

Future versions of this role are intended to better handle these issues.

## WARNINGS

- Do not use this role on production systems. It lacks sufficient testing to
  be relied upon for anything more than local/isolated testing purposes.

## Requirements

- Ubuntu 16.04 or newer
- [LXD packages](https://linuxcontainers.org/lxd/getting-started-cli/)
  installed (tested with 2.0 and 3.0 LTS series)
- Equivalent of `sudo lxd init` already run
  - note: the [`atc0005.lxd-host`
    role](https://github.com/atc0005/ansible-role-lxd-host) role is intended
    to handle this, though as of this writing that role is still incomplete

## Role Variables

- The `state` variable is set outside of the role to act as a control flag for
  either building up a test environment or tearing it down.
  - valid values are `create` or `remove`
- Role variables are set within `default/main.yml`.

Values set here can be overridden in multiple locations, but most commonly via
small include files which reside alongside playbooks or within the playbook
itself. The playbooks found in the
[`ansible-playbook-lxd-testenv`](#references) repo have most of the common
variables listed here directly within the playbooks in an effort to make it
easy to override default values.

### Variables specific to the LXD host

| Variable                                        | Default value               | Purpose                                                                                                                                                                                                                         |
|-------------------------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `state`                                         | `create`                    | Flag with valid values of `create` or `remove`. Used to control whether a test environment will be created or removed/destroyed.                                                                                                |
| `lxd_host_ssh_user_known_hosts_file`            | `$HOME/.ssh/known_hosts`    | Host copy of file specific to user running playbook. Updated to reflect host keys of all containers.                                                                                                                            |
| `lxd_host_ssh_system_known_hosts_file`          | `/etc/ssh/known_hosts`      | Host copy of file for all users. Updated to reflect host keys of all containers.                                                                                                                                                |
| `lxd_host_ssh_client_user_config_file`          | `$HOME/.ssh/config`         | Host copy of file specific to user running playbook. Entries force SSH/Ansible to disable recording and validation of container host keys.                                                                                      |
| `lxd_host_ssh_client_system_config_file`        | `/etc/ssh/ssh_config`       | Host copy of file for all users. Entries force SSH/Ansible to disable recording and validation of container host keys.                                                                                                          |
| `lxd_host_etc_hosts_file`                       | `/etc/hosts`                | This variable exists to allow targeting a hosts file in a non-standard location.                                                                                                                                                |
| `lxd_host_update_user_known_hosts_file`         | `false`                     | Controls whether the `lxd_host_ssh_user_known_hosts_file` file (usually for the user running playbook) is populated with host keys of all containers.                                                                           |
| `lxd_host_update_system_known_hosts_file`       | `false`                     | Controls whether the `lxd_host_ssh_system_known_hosts_file` file (system-wide file for all users) is populated with host keys of all containers.                                                                                |
| `lxd_host_update_ssh_client_user_config_file`   | `false`                     | Controls whether the `lxd_host_ssh_client_user_config_file` file (usually for the user running playbook) is populated with host/ip entries which disable host key checks for matching hosts.                                    |
| `lxd_host_update_ssh_client_system_config_file` | `true`                      | Controls whether the `lxd_host_ssh_client_system_config_file` file (system-wide for all users) is populated with host/ip entries which disable host key checks for matching hosts.                                              |
| `lxd_host_update_etc_hosts_file`                | `true`                      | Controls whether the `lxd_host_etc_hosts_file` file on the LXD container host is updated to include new container name/IP pairs                                                                                                 |
| `lxd_host_stop_containers_before_removal`       | `false`                     | Controls whether an attempt should be made to stop containers before destroying them. Light testing indicates this is not a necessary step, but YMMV.                                                                           |
| `lxd_host_ssh_private_key_for_containers`       | `$HOME/.ssh/id_ed25519`     | Generated if it does not exist. See notes for public key.                                                                                                                                                                       |
| `lxd_host_ssh_public_key_for_containers`        | `$HOME/.ssh/id_ed25519.pub` | Inserted into `root` and `ansible` user profiles within each container.                                                                                                                                                         |
| `lxd_host_remove_container_settings`            | `false`                     | Controls whether this role removes `lxd_host_ssh_known_hosts_file` and `lxd_host_etc_hosts_file` entries as part of teardown tasks. Usually, but not required to be performed along with container removal. Defaults to `false` |
| `lxd_host_remove_containers`                    | `false`                     | Controls whether containers are removed as part of teardown process. Defaults to `false`.                                                                                                                                       |

### Variables specific to LXD containers created by this role

| Variable                                           | Default value                        | Purpose                                                                                                                                                                                                                                                                                                                  |
|----------------------------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `lxd_containers_etc_hosts_file`                    | `/etc/hosts`                         | This variable exists to allow targeting a hosts file in a non-standard location.                                                                                                                                                                                                                                         |
| `lxd_containers_update_hosts_file`                 | `false`                              | Controls whether `lxd_containers_etc_hosts_file` within containers are updated to include entries for *all* containers generated by this role. Since the LXD-provided dnsmasq instance bound to the bridge already handles name resolution, this option is not really needed and may be removed in a future role update. |
| `lxd_containers_image_server`                      | <https://images.linuxcontainers.org> | Server listed in the Ansible module examples. Used to fetch LXD container images for generating fresh containers.                                                                                                                                                                                                        |
| `lxd_containers_create`                            | `true`                               | Controls whether containers are generated when this role is run. **TODO**: Validation is needed to assert that this is not set at the same time as the flag for removing containers.                                                                                                                                     |
| `lxd_containers_bootstrap`                         | `true`                               | Controls whether Python, sudo and other packages are installed as part of preparing new containers for management by Ansible.                                                                                                                                                                                            |
| `lxd_containers_configure`                         | `true`                               | Create service account, groups, enable SSH at boot, etc                                                                                                                                                                                                                                                                  |
| `lxd_containers_packages_required`                 | `openssh-server`, `sudo`             | Packages installed as part of the *bootstrap* process for new containers                                                                                                                                                                                                                                                 |
| `lxd_containers_packages_extra`                    | `nano`                               | Ensure that a sensible (j/k) editor is available in newly created containers                                                                                                                                                                                                                                             |
| `lxd_containers_profiles`                          | `default`                            | LXD profiles applied to all new containers by default.                                                                                                                                                                                                                                                                   |
| `lxd_containers_proxy_server`                      | *empty string*                       | If set, this value is set on containers at the time of creation to allow use of a proxy server. Intended either to permit containers restricted Internet access *or* to allow faster package retrieval via caching proxy.                                                                                                |
| `lxd_containers_docker_config_file_template`       | `docker-daemon-config.json.j2`       | Template file included with role used for setting Docker daemon storage driver.                                                                                                                                                                                                                                          |
| `lxd_containers_docker_storage_driver`             | `vfs`                                | Docker storage drivers: `overlay2` is incompatible with LXD LTS (2.0, 3.0), `overlay` is incompatible with ZFS backed LXD storage. `vfs` is slow, but compatible with both. Override this from a playbook (or other higher level Ansible variable source) if not using ZFS for better performance.                       |
| `lxd_containers_sudoers_include_file`              | `/etc/sudoers.d/ansible`             | Location of sudoers include file used to grant service account full sudo privileges within new containers                                                                                                                                                                                                                |
| `lxd_containers_service_account`                   | `ansible`                            | Service account created within containers that is granted full sudo privileges. Often used by playbooks as a dedicated remote management account.                                                                                                                                                                        |
| `lxd_containers_service_group`                     | `ansible`                            | Service group that goes with the service account already noted here.                                                                                                                                                                                                                                                     |
| `lxc_containers_docker_main_config_file`           | `/etc/docker/daemon.json`            | Path to Docker "daemon" configuration file. Role-provided template for this file configures the active storage driver used within new containers intended to test Docker workflows.                                                                                                                                      |
| `lxc_containers_default_python_install_command`    | *see example below*                  | Crude one-liner shell command to use appropriate package manager to install Python for Ansible use via a `raw` command.                                                                                                                                                                                                  |
| `lxc_containers_python_path`                       | `/usr/bin/python`                    | Location of Python interpreter used by Ansible on remote hosts (containers in our case).                                                                                                                                                                                                                                 |
| `lxd_containers_add_proxy_to_etc_environment_file` | `true`                               | Controls whether `lxd_containers_environment_file` is updated with `http_proxy`, `https_proxy` and `ftp_proxy` environment vars set to the value of `lxd_containers_proxy_server`.                                                                                                                                       |
| `lxd_containers_environment_file`                  | `/etc/environment`                   | If `lxd_containers_add_proxy_to_etc_environment_file` is enabled, then the standard `http_proxy`, `https_proxy` and `ftp_proxy` environment variables are set within this file.                                                                                                                                          |


Current shell command for the `lxc_containers_default_python_install_command` variable:

```shell
if [ -f /usr/bin/apt-get ]; then
  apt-get install -y python;
fi
if [ -f /usr/bin/yum ]; then
  yum install -y python;
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

  # Rely on this setting to be specified in host_vars or group_vars files. If
  # we lock in the 'local' option here then that rules out setting up any
  # remote hosts as LXD hosts.
  # connection: local

  # Should be fine to gather facts for just the future LXD host
  gather_facts: yes

  tasks:

    # Stub role for now. Later work will have it install required packages
    # needed in order to configure host system for running LXD containers
    - name: Apply lxd-host role to install LXD packages, perform initial setup
      import_role:
        name: atc0005.lxd-host

- name: Setup LXD test environment
  hosts: localhost

  # Rely on this setting to be specified in host_vars or group_vars files. If
  # we lock in the 'local' option here then that rules out setting up any
  # remote hosts as Docker hosts.
  # connection: local

  # This is handled by the earlier atc0005.lxd-host role play
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

        # Note: Other values can be set here to override role defaults.
        # See the README file for details.

```

### Teardown/Remove test environment

**WARNING:** Review known [Limitations](#limitations) before using this role.

```yaml

- name: Tear down our LXD test environment
  hosts: localhost

  # Rely on this setting to be specified in host_vars or group_vars files. If
  # we lock in the 'local' option here then that rules out connecting via SSH
  # later, perhaps as a means of validating earlier playbook tasks.
  # connection: local

  # Fact gathering is not needed for container and settings removal
  gather_facts: no

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

Note: This role is not yet available via Ansible Galaxy, so for now you will
need to install it via a `requirements.yml` file.

#### Specific stable release version

```yaml
---

# This stub role is referenced by the lxd-testenv setup playbook,
# so we're using an early tag/release to satisfy those requirements
- name: "atc0005.lxd-host"
  src: https://github.com/atc0005/ansible-role-lxd-host.git"
  version: "v0.1-alpha"

- name: "atc0005.lxd-testenv"
  src: https://github.com/atc0005/ansible-role-lxd-testenv.git"
  version: "v1.1"

...

```

#### Latest from the `master` branch

Example `requirements.yml` file that uses the latest from the `master` branch:

```yaml
---

# This stub role is referenced by the lxd-testenv setup playbook,
# so we're including that there to satisfy those requirements.
# Referencing the master branch for this role to match the same branch
# reference for the atc0005.lxd-testenv repo.
- name: "atc0005.lxd-host"
  src: https://github.com/atc0005/ansible-role-lxd-host.git"
  version: "master"

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

- <https://github.com/atc0005/ansible-playbook-lxd-testenv>

- <https://github.com/atc0005/ansible-playbooks>
- <https://github.com/atc0005/ansible-role-lxd-host>
- <https://github.com/atc0005/ansible-role-lxd-testenv>
- <https://github.com/atc0005/ansible-role-docker-host>

- <https://linuxcontainers.org/lxd/getting-started-cli/>

- <https://docs.ansible.com/ansible/latest/modules/lxd_container_module.html>
- <https://docs.ansible.com/ansible/latest/modules/lxd_profile_module.html>
