Know Issues
-----------

In VirtualBox subsystem, NVMe causes errors in DM Multipath.

In VirtualBox subsystem, LVM and DM Multipath expects a different WWID so you must make manual adjustments.

See **1ATA_** replacements:

  ```shell
  tasks/setup/boot-from-san.yml:60: wwid {{ fact_real_boot_disk_wwid | regex_replace('1ATA_') if ansible_facts['service_mgr'] == 'systemd' else fact_real_boot_disk_wwid }}
  tasks/setup/boot-from-san.yml:112: command: "multipath -a {{ fact_real_boot_disk_wwid | regex_replace('1ATA_') if ansible_facts['service_mgr'] == 'systemd' else fact_real_boot_disk_wwid }}"
  tasks/setup/local-boot.yml:56: wwid {{ fact_real_boot_disk_wwid | regex_replace('1ATA_') }}
  tasks/setup/local-boot.yml:140: multipath -v 4 -r -d 2>&1 | grep '{{ fact_real_boot_disk_wwid | regex_replace('1ATA_') }}' | grep 'blacklisted'
  tasks/presetup/local-boot/presetup-local-lvm.yml:19: - '{{ fact_lvm_filter_parameter }} = [ "a/{{ fact_real_boot_disk_wwid | regex_replace("1ATA_") }}/", "r/block/", "r/disk/", "r/sd.*/", "a/.*/" ]'
  tasks/presetup/local-boot/presetup-local-lvm.yml:26: - fact_lvm_filter == '{{ fact_lvm_filter_parameter }} = [ "a/{{ fact_real_boot_disk_wwid | regex_replace("1ATA_") }}/", "r/block/", "r/disk/", "r/sd.*/", "a/.*/" ]'
  templates/localboot_multipath.conf.j2:103: wwid {{ fact_real_boot_disk_wwid | regex_replace('1ATA_') }}
  templates/bootfromsan_multipath.conf.j2:107: wwid {{ fact_real_boot_disk_wwid | regex_replace('1ATA_') if ansible_facts['service_mgr'] == 'systemd' else fact_real_boot_disk_wwid }}
  ```

Playbook using sshpass (ansible_password) and Python3, causes errors in synchronize module when used with delegate_to ([Errno 32] Broken pipe), use SSH Key based authentication instead.
