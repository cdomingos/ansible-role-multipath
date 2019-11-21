Ansible Role: Multipath (DM Multipath)
======================================

A fine ansible role which deploy and setup linux native multipathing software (DM Multipath) properly for *Boot from SAN* or *Local Boot* scenarios.

This **multipath** ansible role is able to:

- configure DM Multipath according to product documentation and industry best practices
- configure driver parameters

## Warning

**This role replaces the DM Multipath configuration**  
Previous settings will be overwrite, check `multipath_backup_permanent` in Role Variables section.

This configuration affects the following directory and configuration files:

* `/etc/multipath/`
* `/etc/multipath.conf`

<br>

**If something is wrong after reboot the managed host, some errors in initramfs can leads to [`Kernel panic on boot following "dracut Warning: LVM rootvg/rootlv not found"`](https://access.redhat.com/solutions/1282013)**

In this case, I recommend:

1. reboot the system using a Rescue ISO Image
2. revert back main files:
   - /boot/\<original-initramfs\>.img.ansible_multipath
   - /etc/fstab.ansible_multipath
   - /etc/lvm/lvm.conf.ansible_multipath

NOTE: there are many ways to revert your system to operational state, kernel boot parameters as `rdshell` or `rd.shell` plus removing `rhgb` and `quiet` can help you to troubleshoot your broken initramfs.


How it works?
-------------

There are two main stages: **pre-setup** and **setup**.

In pre-setup stage this role does (no system changes, except for package installation if declared):

  - check if device-mapper-multipath package is installed (you can install or upgrade it. See variables)
  - get properties of device for /boot mounted filesystem, also for /boot/efi
  - get properties and discover the real disk device for /boot mounted filesystem
  - load modprobe drivers according vendor and model from discovered real boot disk device
  - autoselect the type of configuration *(Boot from SAN?)* based on boot disk device driver
  - get GRUB's default kernel and initramfs
  - load verifications for Boot from SAN or Local Boot scenarios
  - defines pseudo name for real disk device
  - defines LVM filter

In setup stage this role does (system changes):

  - backup some system files, such as /etc/fstab, current initramfs, and /etc/lvm/lvm.conf
  - removes current DM Multipath configuration files
  - create an initial /etc/multipath.conf file
  - setup /etc/multipath.conf file and check for syntax or errors
  - setup /etc/fstab
  - setup LVM filter and check for syntax or errors
  - setup modprobe files and check for syntax or errors
  - setup initramfs and check for errors
  - in case of errors, it removes extra files and rollback system files using previous backup
  - reboot the system
  - removes previous backup system files


Requirements
------------

Playbook must run with `become: yes` or equivalent root privilege and `gather_facts: yes` as well.

- Ansible Engine 2.7.1+


Role Variables
--------------

The **multipath** role is configured over variables starting with `multipath_` as the name prefix.

List of variables:

| Variable                              | Type         | Description                                                                              |
|---------------------------------------|--------------|------------------------------------------------------------------------------------------|
| `multipath_setup_pseudodevname`       | string (m)  *| Define pseudo device name (alias) for boot disk managed by DM Multipath                  |
| `multipath_package_install`           | boolean (o)  | Install the device-mapper-multipath package if not present                               |
| `multipath_package_upgrade`           | boolean (r)  | Upgrade (also install) device-mapper-multipath package                                   |
| `multipath_preserve_currentfiles`     | boolean (r)  | Before setup stage, make a permanent copy of current DM Multipath configuration files    |
| `multipath_reboot_allow`              | boolean (r)  | Allow this role to reboot the managed node                                               |
| `multipath_reboot_timeoutguest`       | integer (o)  | Max time (secs) for 'reboot' waiting a managed Guest system to becomes online again      |
| `multipath_reboot_timeoutbaremetal`   | integer (o)  | Max time (secs) for 'reboot' waiting a managed Bare Metal system to becomes online again |
| `multipath_modprobe_custom`           | list (o)     | Load and define drivers parameters related to boot device disk vendor or model           |
| `multipath_configfile_commentlines`   | boolean (o) *| Comment all lines in /etc/multipath.conf                                                 |
| `multipath_configfile_bybasic`        | boolean (o) *| Overwrites /etc/multipath.conf file using a basic and functional configuration           |
| `multipath_configfile_bytemplate`     | boolean (o)  | Overwrites /etc/multipath.conf file using predefined role templates                      |
| `multipath_configfile_byusertemplate` | string (o)   | Overwrites /etc/multipath.conf file using a custom user template file                    |
| `multipath_configfile_bysections`     | string (o)   | Include custom sections in /etc/multipath.conf file                                      |
| `multipath_configfile_defaults`       | list (o)     | Include extra parameters in "defaults {" section for /etc/multipath.conf                 |
| `multipath_debug_sysfilecontent`      | boolean (o)  | Before post-setup tasks (e.g. reboot), show content of modified system files             |
| `multipath_debug_showvariables`       | boolean (r)  | Show role variables and their values                                                     |

(m) - mandatory  
(r) - recommended  
(o) - optional  
\* - default value is applied ( see `defaults/main.yml` )

Explained list of variables:

* `multipath_setup_pseudodevname`: define pseudo device name (alias) for boot disk managed by DM Multipath (e.g. /dev/mapper/bootLUN)

  type: string

  affected files:
    - /etc/fstab
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_setup_pseudodevname: bootLUN
  ```

  NOTE: do not use "p" or numeric characteres

<br>

* `multipath_package_install`: install the device-mapper-multipath package if not present

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_package_install: yes
  ```

<br>

* `multipath_package_upgrade`: upgrade (also install) the device-mapper-multipath package

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_package_upgrade: yes
  ```

<br>

* `multipath_preserve_currentfiles`: before setup stage, make a permanent copy of current DM Multipath configuration files, such as /etc/multipath.conf and /etc/multipath/

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_preserve_currentfiles: yes
  ```

  NOTE: preserved files will be stored as /etc/multipath.conf.ansible_multipath and /etc/multipath.ansible_multipath/, once these files are copied the role will never touch them again, for directory it acts as append mode.

<br>

* `multipath_reboot_allow`: allow this role to reboot the managed node

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_reboot_allow: yes
  ```

<br>

* `multipath_reboot_timeoutguest`: max time (secs) for 'reboot' waiting a managed Guest system to becomes online again

  type: integer

  affected files: n/a

  example:

  ```yaml
  multipath_reboot_timeoutguest: 300
  ```

<br>

* `multipath_reboot_timeoutbaremetal`: max time (secs) for 'reboot' waiting a managed Bare Metal system to becomes online again

  type: integer

  affected files: n/a

  example:

  ```yaml
  multipath_reboot_timeoutbaremetal: 1800
  ```

<br>

* `multipath_modprobe_custom`: load and define drivers parameters related to boot device disk vendor or model, generating a modprobe file according to module name. - please refer your connectivity guide about Enterprise Storage Array vendor

  type: list

  affected files:
    - /etc/modprobe.d/\<module-name\>.conf

  example:

  ```yaml
  multipath_modprobe_custom:
    - module: lpfc
      params: lpfc_max_luns=65535 lpfc_devloss_tmo=10
      vendor: EMC
      model: any
    - module: lpfc
      params: lpfc_max_luns=65535 lpfc_devloss_tmo=14 lpfc_lun_queue_depth=16 lpfc_discovery_threads=32
      vendor: 3PARdata
      model: VV
    - module: qla2xxx
      params: ql2xmaxlun=65535 qlport_down_retry=14 ql2xmaxqdepth=16
      vendor: 3PARdata
      model: VV
    - module: scsi_transport_fc
      params: dev_loss_tmo=10
      vendor: any
      model: any
  ```

  NOTE: using **"any"** value will adjust driver parameters without matching/validates boot disk device driver, you can obtain a full device list through `multipath -t`

<br>

* `multipath_configfile_commentlines`: comment all lines in /etc/multipath.conf after mpathconf tool creates an initial configuration file

  type: boolean

  affected files:
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_configfile_commentlines: yes
  ```

<br>

* `multipath_configfile_bybasic`: overwrites /etc/multipath.conf file using a basic and functional configuration (original file comments are not preserved)

  type: boolean

  affected files:
    - /etc/multipath.conf

  dependencies:
    - `multipath_configfile_commentlines: yes`

  example:

  ```yaml
  multipath_configfile_bybasic: yes
  ```

<br>

* `multipath_configfile_bytemplate`: overwrites /etc/multipath.conf file using predefined role templates (e.g. templates/bootfromsan_multipath.conf.j2) with basic functional configuration (original file comments are preserved)

  type: boolean

  affected files:
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_configfile_bytemplate: yes
  ```

<br>

* `multipath_configfile_byusertemplate`: overwrites /etc/multipath.conf file using a custom user template file, the file must be at playbook directory level or 'templates' as well, likewise you can use role facts, see 'templates/bootfromsan_multipath.conf.j2' file

  type: string (file-path)

  affected files:
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_configfile_byusertemplate: mymultipath.conf.j2
  ```

<br>

* `multipath_configfile_bysections`: include custom sections in /etc/multipath.conf file

  type: string (multi-line)

  affected files:
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_configfile_bysections: |

    devices {
            device {
                    vendor                   "COMPAQ  "
                    product                  "HSV110 (C)COMPAQ"
                    path_grouping_policy     multibus
                    path_checker             readsector0
                    path_selector            "round-robin 0"
                    hardware_handler         "0"
                    failback                 15
                    rr_weight                priorities
                    no_path_retry            queue
            }
    }
  ```

  NOTE: optional combined with `multipath_configfile_commentlines: yes`

<br>

* `multipath_configfile_defaults`: include extra parameters in "defaults {" section for /etc/multipath.conf

  type: list

  affected files:
    - /etc/multipath.conf

  example:

  ```yaml
  multipath_configfile_defaults:
    - detect_prio no
    - failback "manual"
    - max_fds 1048576
    - max_polling_interval 20
  ```

<br>

* `multipath_debug_sysfilecontent`: before post-setup tasks (e.g. reboot), show content of modified system files in playbook stream like a task, files: /etc/fstab, /etc/lvm/lvm.conf and /etc/multipath.conf

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_debug_sysfilecontent: yes
  ```

<br>

* `multipath_debug_showvariables`: show role variables and their values

  type: boolean

  affected files: n/a

  example:

  ```yaml
  multipath_debug_showvariables: yes
  ```

Dependencies
------------

N/A


Tags
----

There are two available tags:

* `role:multipath:prereqs`: role prerequisites, check role variables, no system changes

  ```shell
  ansible-playbook multipathing.yml --tags role:multipath:prereqs
  ```

* `role:multipath:presetup`: pre-setup validations, no system changes (except for requested to install or upgrade DM Multipath package using **multipath\_cfg\_package_install** or **multipath\_cfg\_package_upgrade** variables)

  ```shell
  ansible-playbook multipathing.yml --tags role:multipath:presetup
  ```


Example Playbook
----------------

A basic playbook should be like this:

    - hosts: servers
      become: yes
      gather_facts: yes

      roles:
        - cdomingos.multipath

List of example playbooks can be found at `tests/` in role directory structure.


License
-------

MIT


Author Information
------------------

This role was created in 2019 by [Cláudio Domingos](https://linkedin.com/in/cadomingos)


Additional Information
----------------------

Tested on:  
  - HP ProLiant BL460c Gen9 (UEFI)  
  - 3PAR Storage Array  
  - HP ProLiant BL460c G7  
  - VMAX Storage Array  
  - RHEL 8  
  - RHEL 7  
  - RHEL 6
  - Fedora 31  

It is very appropriate to use this role for Bare Metal machines, guests does not make sense because high availability multipathing software must be at hypervisor level.

*Do you want to apply DM Multipath best practices configuration using an specific Storage Array?* Engage your Storage Array vendor, also you can searching for **Host Connectivity Guide for Linux** and check what your Storage Array vendor recommends for.

This role is inspired from:  
  - [**Red Hat Enterprise Linux 8 Configuring Device Mapper Multipath**](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_device_mapper_multipath/index) guide  
  - [**Red Hat Enterprise Linux 7 Configuring Device Mapper Multipath**](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html-single/dm_multipath/index) guide  
  - [**Red Hat Enterprise Linux 6 Configuring Device Mapper Multipath**](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html-single/dm_multipath/index) guide


TODO
----

Solve critical issue like `dracut Warning: LVM rootvg/rootlv not found`

a. Implementing a script to rollback at initramfs/dracut boot stage when system crash during reboot cycle  
b. Enable network and sshd service at initramfs/dracut boot stage and perform necessary rescue tasks to fix it using Ansible
