[![Build Status](
https://travis-ci.org/nickrusso42518/racc.svg?branch=master)](
https://travis-ci.org/nickrusso42518/racc)

# Run Arbitrary CLI Commands (racc)
This playbook runs arbitrary commands on a given node, stores the
output in a flat text file, and archives the entire "run" in an archive
file for easy copying to management stations via SCP. It is generally used
for configuration backups, hardware inventory, and license information.

> Contact information:\
> Email:    njrusmc@gmail.com\
> Twitter:  @nickrusso42518

  * [Supported platforms](#supported-platforms)
  * [Variables](#variables)
  * [Task summary](#task-summary)
  * [Output files](#output-files)

## Supported platforms
Any host with an Ansible `*_command` module can be supported, including
non-Cisco devices. The playbook currently provides Ansible task files for
Cisco IOS/IOS-XE, Cisco IOS-XR, Cisco ASA, Cisco NX-OS, and Cisco AireOS
device types. To add a new device type, create a new task file in the
`tasks/` folder along with a corresponding `group_vars/` and inventory entry.
These are all easy tasks given the abstract architecture of this playbook.

Testing was conducted on the following platforms and versions:
  * Cisco CSR1000v, version 16.08.01a, running in AWS
  * Cisco CSR1000v, version 16.09.02, running in AWS
  * Cisco XRv9000, version 6.3.1, running in AWS
  * Cisco ASAv, version 9.9.1, running in AWS
  * Cisco Nexus 3172T, version 6.0.2.U6.4a, hardware appliance
  * Cisco vWLC, version 8.3.143.0, running on VMware ESXi

```
$ cat /etc/redhat-release
Red Hat Enterprise Linux Server release 7.4 (Maipo)

$ uname -a
Linux ip-10-125-0-100.ec2.internal 3.10.0-693.el7.x86_64 #1 SMP
  Thu Jul 6 19:56:57 EDT 2017 x86_64 x86_64 x86_64 GNU/Linux

$ ansible --version
ansible 2.6.2
  config file = /home/ec2-user/natm/ansible.cfg
  configured module search path =
    ['/home/ec2-user/.ansible/plugins/modules',
     '/usr/share/ansible/plugins/modules']
  ansible python module location =
    /home/ec2-user/environments/a262/lib64/python3.6/site-packages/ansible
  executable location = /home/ec2-user/environments/a262/bin/ansible
  python version = 3.6.7 (default, Dec  5 2018, 15:02:05)
    [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

## Variables
The `group_vars/all.yml` file contains connectivity parameters common to all
network modules. These typically use `network_cli` but for modules not yet
updated (or for users running older versions of Ansible), a legacy provider
structure is included as well. Note that if the legacy method is used, the
individual task files for each platform must set `ansible_connection: local`
in addition to manually specifying `provider: "{{ legacy_creds }}"` as a task
suboption in most cases.

There are some minor variables that control the playbooks output products
and formatting. These are not often modified:
  * `remove_files`: A boolean true/false question that governs whether items
    in the `files/` folder are removed after a playbook run. Normally, this
    is set to `true` because the final archive is what really matter, and
    retaining all the uncompressed text files on the control machine is not
    a long-term solution. Setting this to `false` is useful for quick tests
    or troubleshooting.
  * `archive_format`: Determines what kind of archive to create when the
    `archive` moduel is called. Any option supported by the version of Ansible
    in use can be used here. Some examples include zip, gz, bz2, xz, and tar
    as of Ansible 2.5. The zip format is usually appropriate when transferring
    to Windows-based SCP servers, and that is the default.
  * `scp`: Nested dictionary containing two subkeys below.
    * `user`: The SCP username that can write files to the SCP server.
    * `host`: The SCP server FQDN/IP address. Under its root directory, a
      directory called `racc/` should be created. This is where the archives
      generated by this playbook are copied. This playdoes __does not__
      automatically perform the copying.

There are independent `group_vars/` files for each network device type:
routers, switches, and firewalls, and wireless LAN controllers for example.
Each one has a key of `command_list` which specifies a list of dictionaries
as shown below. The `file_suffix` value specifies what is written to the
end of the file name, and the `command` value is the literal command to
be issued to the device. CLI output redirection (pipes) are supported.
It is recommended to type the entire command into the `command` value
since these are printed to stdout during execution. Using abbreviated
commands may confuse operators less familiar with some network CLIs.

Note that these are just examples and it is common to adjust the
`command_list` on a per-run basis. For example, if collecting all routing
tables is important to quickly troubleshoot a routing loop, simply add
`show ip route` with a corresponding file suffix to the list, and run
the playbook.

```
$ cat group_vars/ios_router.yml
---
ansible_network_os: "ios"
command_list:
  - file_suffix: "cfg"
    command: "show running-config"
  - file_suffix: "inv"
    command: "show inventory"
  - file_suffix: "lic"
    command: "show license all"
  - file_suffix: "ver"
    command: "show version"
...

$ cat group_vars/aireos_wlc.yml
---
ansible_connection: "local"
ansible_network_os: "aireos"
command_list:
  - file_suffix: "wln"
    command: "show wlan summary"
  - file_suffix: "inv"
    command: "show inventory"
  - file_suffix: "lic"
    command: "show license all"
  - file_suffix: "ver"
    command: "show boot"
...
```

## Task summary
To make the playbook easier to read and troubleshoot, whenever a command is
issued on a device, both the inventory hostname and the command being issued
are printed to stdout. The example below shows a sample run with a variety
of devices.

```
TASK [ASA >> Gather Cisco ASA information]
ok: [asav1] => (item=show running-config)
ok: [asav1] => (item=show inventory)
ok: [asav1] => (item=show version)

TASK [IOSXR >> Gather Cisco IOS-XR information]
ok: [xrv1] => (item=show running-config)
ok: [xrv1] => (item=show inventory)
ok: [xrv1] => (item=show license all)
ok: [xrv1] => (item=show version)

TASK [IOS >> Gather Cisco IOS/IOS-XE information]
ok: [csr1] => (item=show running-config)
ok: [csr2] => (item=show running-config)
ok: [csr1] => (item=show inventory)
ok: [csr2] => (item=show inventory)
ok: [csr1] => (item=show license all)
ok: [csr2] => (item=show license all)
ok: [csr1] => (item=show version)
ok: [csr2] => (item=show version)
```

## Output files
At the end of the playbook, assuming that `remove_files` is set to
`false` for the purpose of discussion, the following filesystem
components are created.

For each host against which this playbook runs, N files are generated,
where N is the number of elements in the relevant `command_list`.
Ansible inventory hostname and the date/time group (DTG) in UTC are
included in the file names for easy identification. All files are written
to `files/` directory and are ignored by git. The format of all text files is
`<hostname>_<dtg>_<file_suffix>.txt` while the parent directory format is
`<hostname>_<dtg>/`.

```
$ tree files/
files/
├── asav1_20180603T070140
│   ├── asav1_20180603T070140_cfg.txt
│   ├── asav1_20180603T070140_inv.txt
│   └── asav1_20180603T070140_ver.txt
├── csr1_20180603T070140
│   ├── csr1_20180603T070140_cfg.txt
│   ├── csr1_20180603T070140_inv.txt
│   ├── csr1_20180603T070140_lic.txt
│   └── csr1_20180603T070140_ver.txt
├── csr2_20180603T070140
│   ├── csr2_20180603T070140_cfg.txt
│   ├── csr2_20180603T070140_inv.txt
│   ├── csr2_20180603T070140_lic.txt
│   └── csr2_20180603T070140_ver.txt
└── xrv1_20180603T070140
    ├── xrv1_20180603T070140_cfg.txt
    ├── xrv1_20180603T070140_inv.txt
    ├── xrv1_20180603T070140_lic.txt
    └── xrv1_20180603T070140_ver.txt
```

The actual text output is shown below. The exclamation-mark delimeters are
useful when concatenating multiple files together since determining where
one output ends and another starts is difficult otherwise.

```
$ cat files/csr1_20180603T070140/*
[snip, running-config output omitted for brevity]
!!!
!!!   END OUTPUT FROM: show running-config
!!!
!!!
!!! BEGIN OUTPUT FROM: show inventory
!!!
NAME: "Chassis", DESCR: "Cisco CSR1000V Chassis"
PID: CSR1000V          , VID: V00  , SN: 9VCVGDCB4JW

NAME: "module R0", DESCR: "Cisco CSR1000V Route Processor"
PID: CSR1000V          , VID: V00  , SN: JAB1303001C

NAME: "module F0", DESCR: "Cisco CSR1000V Embedded Services Processor"
PID: CSR1000V          , VID:      , SN:
!!!
!!!   END OUTPUT FROM: show inventory
!!!
!!!
!!! BEGIN OUTPUT FROM: show license all
!!!
License Store: Primary License Storage
!!!
!!!   END OUTPUT FROM: show license all
!!!
!!!
!!! BEGIN OUTPUT FROM: show version
!!!
Cisco IOS XE Software, Version 16.08.01a
[snip, extra output omitted for brevity]
```

Finally, the playbook prints out a shell command that can be copy/pasted by
the user to quickly SCP the archive to an SCP server, assuming an FQDN/IP
has been specified for it. This is useful for those unfamiliar with the
Linux `scp` command.

`scp archives/commands_20180603T070140.zip nick@192.0.2.1:/racc/`
