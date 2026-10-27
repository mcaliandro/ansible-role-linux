Ansible role for Linux hosts
=========

This role provides a generic management of Linux hosts:
- setup a hostname;
- installation and removal of packages;
- create partitions using `parted` and setup automount using `fstab`;
- disable active swap;
- create and configure swap file;
- setup kernel settings using `sysctl`;
- load and blacklist kernel modules using `modprobe`.


Requirements
------------

From collection [ansible.posix](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/index.html), version 2.2.2:
- [mount](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/mount_module.html) module to configure mount points in `fstab`;
- [sysctl](https://docs.ansible.com/projects/ansible/latest/collections/ansible/posix/sysctl_module.html) module to manage the kernel paramaters.

From collection [community.general](https://docs.ansible.com/projects/ansible/latest/collections/community/general/index.html), version 13.4.0:
- [filesystem](https://docs.ansible.com/projects/ansible/latest/collections/community/general/filesystem_module.html) module to create filesystems;
- [modprobe](https://docs.ansible.com/projects/ansible/latest/collections/community/general/modprobe_module.html) module to manage kernel modules;
- [parted](https://docs.ansible.com/projects/ansible/latest/collections/community/general/parted_module.html) to create partitions using `parted`.


Role Variables
--------------

A description of the settable variables for this role (see `defaults/main.yml` and `vars/main.yml`).

### Force reboot
Setup `linux_force_reboot` variable in order to reboot the machine right when it finishes its execution.
By default, this role reboots the machine only when `Reboot` handler is notified by some task.
```yaml
linux_force_reboot: false
```

### Setup hostname
Setup `linux_hostname` variable to configure the hostname on the host.
By default, the variable is set to "empty", so the roles does not try to change the hostname currently set on the host.
```yaml
linux_hostname: ""
```

### Remove snapd (for Ubuntu only)
Setup `linux_remove_snapd` variable to completely remove software installed with Snap and Snap (daemon) itself from a Ubuntu host.
By default, the roles keeps Snap daemon and the snaps installed on the host.
```yaml
linux_remove_snapd: false
```

### Manage packages
Use the list `linux_packages` to specify the packages to be installed or removed.
This role uses [ansible.builtin.package](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/package_module.html) module to manage packages without the need to specify the package manager module (`apt`, `dnf`, etc.).

For example:
```yaml
linux_packages:
  - python3
  - python3-pip
```

If you need to remove a package, you need to specify both `name=PACKAGE_NAME` and `state=absent` fields.
In this example, the role removes `httpd` package first and then installs `nginx` package:
```yaml
linux_packages:
  - name: httpd
    state: absent
  - nginx
```

### Configure OpenSSH server (`sshd_config`)
Use the dictionary `linux_sshd_config` to specify the sshd parameters and their values as key-value entries.
These entries will be stored into the default `sshd_config` file, which path is `/etc/ssh/sshd_config` (see the variable `linux_sshd_default_config_file` in `vars/main.yml`).

For example:
```yaml
linux_sshd_config:
  PermitRootLogin: "no"
  PasswordAuthentication: "no"
  PubkeyAuthentication: "yes"
```

If you prefer to upload `sshd_config` dropin files to the remote host, specify them using the list `linux_sshd_config_dropin_files`.
You can store your dropin files within `files/` directory of this role, or in a sub-path of your Ansible project's `inventory_dir` path.
Dropin files on the remote host can be removed by specifying both `file=FILENAME` and `state=absent` fields.

In this example, custom dropin file `00-override.conf` is uploaded into `/etc/ssh/sshd_config.d/` directory, while `50-cloud-init.conf` is removed from the same directory:
```yaml
linux_sshd_config_dropin_files:
  - "{{ inventory_dir }}/host_vars/{{ inventory_hostname }}/sshd_config.d/00-override.conf"
  - file: "50-cloud-init.conf"
    state: absent
```

### Manage storage: disks and partitions
Use the `linux_disks` list to specify the partitioning structure to be applied on each attached disk.

This example reports the complete list of mandatory and optional fields:
```yaml
linux_disks:
  - device: /dev/sdb  
    part_table_type: gpt
    partitions:
      - name: data
        number: 1
        start: 0%
        end: 100%
        fs_type: ext4
        fs_opts: -b 2048
        automount: true
        mount_path: /mnt/data
        mount_opts: defaults
```

For each *linux_disks* entry,  
- `device` (mandatory): the path of the device attached to the host, for example `/dev/sdb`;
- `part_table_type` (optional): the type of partition table used on the disk, accepted values: `gpt` (default), `msdos` (see the variable `linux_supported_partition_table_types` defined in `vars/main.yml`);
- `partitions` (mandatory): the list of partitions to create on the disk.

For each *partitions* entry:
- `name` (mandatory): the name to apply as label of the filesystem, for example `volume`;
- `number` (mandatory): the number of partition device (must be incremental in case of multiple partitions on the disk);
- `start` (optional): see `community.general.parted` module's parameter [part_start](https://docs.ansible.com/projects/ansible/latest/collections/community/general/parted_module.html#parameter-part_start);
- `end` (optional): see `community.general.parted` module's parameter [part_end](https://docs.ansible.com/projects/ansible/latest/collections/community/general/parted_module.html#parameter-part_end);
- `fs_type` (mandatory): which filesystem type the partition must be formatted with, accepted values: `btrfs`, `ext2`, `ext4`, `vfat` (see the variable `linux_supported_partition_filesystems` defined in `vars/main.yml`);
- `fs_opts` (optional): the options to be passed to `mkfs` command;
- `automount` (optional): whether to add an entry in fstab (`automount=true`, default) or not (`automount=false`);
- `mount_path` (required if `automount=true`): the mount point of the partition in fstab;
- `mount_opts`: pass the list of comma-separated mount options to fstab.

A minimum required configuration is the following:
```yaml
linux_disks:
  - device: /dev/sdb
    partitions:
      - name: data
        number: 1
        fs_type: ext4
        mount_path: /mnt/data
```

### Disable swap
Setup the variable `linux_disable_swap=true` in order to disable the swap space on the host. This role does not free or remove any configured swap partition.

By default, the value is set to `false`, so the role does not change the current swap configuration (whether it is active or not) on the host:
```yaml
linux_disable_swap: false
```

### Manage a swap file
Use the dictionary `linux_swapfile` to create and configure a swap file on the host.

**IMPORTANT**: This requires to setup variable `linux_disable_swap=false` (default), otherwise the configuration tasks will be skipped.

Here, a description of the supported fields:
- `path` (optional): absolute path to the swap file; if not defined, the default path `/swapfile` is used (see the variable `linux_swapfile_default_path` in `vars/main.yml`);
- `size_mb` (mandatory): size in MB of the swap file;
- `mount_opts` (optional): specify the fstab options; if not defined, fstab entry is configured with `defaults` option only;
- `state` (optional): whether the swap file should be created (`state=present`, default) or not (`state=absent`).

A minimum required configuration is the following:
```yaml
linux_swapfile:
  size_mb: 4096
```

**NOTE**: When `linux_disable_swap=true`, the swap file exists and `linux_swapfile.state=absent`, the role disables the swap space as planned and removes also the swap file.

To remove the swap file from the host:
```yaml
linux_disable_swap: true
linux_swapfile:
  state: absent
```

### Manage kernel modules (`modprobe`)
Use the list `linux_modprobe_load` to specify the kernel modules to be loaded on the host.

For example:
```yaml
linux_modprobe_load:
  - overlay
  - br_netfilter
```

If you need to specify also the module's parameters, use the field `params` (see `community.general.modprobe` module's parameter called [params](https://docs.ansible.com/projects/ansible/latest/collections/community/general/modprobe_module.html#parameter-params)).

For example:
```yaml
linux_modprobe_load:
  - name: MODULE_NAME
    params: "SPACE_SEPARATED_PARAMS"
```

Use the list `linux_modprobe_blacklist` to specify the kernel modules to be loaded on the host.

For example:
```yaml
linux_modprobe_blacklist:
  - usb-storage
```

### Manage kernel settings (`sysctl`)
Use the dictionary `linux_sysctl_params` to specify the kernel parameters and their values as key-value entries.
These entries will be stored into the default `sysctl.conf` file, which path is `/etc/sysctl.conf` (see the variable `linux_sysctl_default_config_file` in `vars/main.yml`).

For example:
```yaml
linux_sysctl_params:
  net.ipv4.ip_forward: "1"
  net.ipv4.conf.all.forwarding: "1"
  net.ipv6.conf.all.forwarding: "1"
```

If your prefer to upload custom `sysctl` dropin files to the remote host, specify them using the list `linux_sysctl_config_dropin_files`.
You can store your dropin files within `files/` directory of this role, or in a sub-path of your Ansible project's `inventory_dir` path.
Dropin files on the remote host can be removed by specifying both `file=FILENAME` and `state=absent` fields.

In this example, custom dropin file `10-security.conf` is uploaded into `/etc/sysctl.d/` directory, while `99-tuning.conf` is removed from the same directory:
```yaml
linux_sysctl_config_dropin_files:
  - "{{ inventory_dir }}/host_vars/{{ inventory_hostname }}/sysctl.d/10-security.conf"
  - file: "99-tuning.conf"
    state: absent
```


Dependencies
------------

None.


Example Playbook
----------------

```yaml
# Usage:
#   ansible-playbook -l <host/group> linux.yml
---
- name: Manage Linux hosts
  hosts: all
  become: true

  roles:
    - mcaliandro.linux
```


Compatibility
-------------

This role has been tested on these Linux distributions:
- Ubuntu Server LTS 22.04
- Ubuntu Server LTS 24.04
- Ubuntu Server LTS 26.04


License
-------

Apache License


Author Information
------------------

Created in 2026 by Matteo Caliandro, <mcaliandro.dev@gmail.com>
