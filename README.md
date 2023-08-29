# Hardening SELinux
SELinux is a flexible and powerful technology, which can be easily configured to mitigate the technical impacts of many weaknesses and vulnerabilities. SELinux is shipped with a default configuration that does not require ordinary user intervention. However, usability comes at the expense of security. SELinux has a lot of useful security features and protections that can be used to harden security, without limitations.

## Installation
The Ansible tool has to be installed to configure and apply playbooks.  For SELinux configuration, Ansible also needs the SELinux role to be installed.  All necessary packages are installed by following commands:

### Fedora
```
# dnf install ansible
# ansible-galaxy install linux-system-roles.selinux
```

### RHEL9
```
# dnf install ansible*
# ansible-galaxy install linux-system-roles.selinux
# ansible-galaxy collection install community.general ansible.posix

```

Download SELinux configuration in Ansible:

```
$ git clone https://gitlab.com/5umm3r15/selinux-hardening.git
```
## Configuration

### Hosts
Currently, a configuration is targeted on the localhost.  In the directory inventory is the file hosts.yaml, in which can be defined hosts.
```sh
[group]
address
```
For example:
```sh
[webservers]
192.168.122.164
```
When managing multiple hosts, it is necessary to define them to the **hosts** in the main configuration file selinux-playbook.yml and use the `-i inventory/hosts` option when running a playbook.



### Users

Linux users are mapped into SELinux policy in file the /users/user_mapping.yml. 
- After **login** is written *username*, that will be confined by SELinux user. 
- The *SELinux user* is defined after **seuser**. 
- When **state** has the value: 
	- *present* - the user is mapped into SELinux users, 
	- *absent* - the user is removed from the policy.
```sh
# Usage:
user_mapping:
  - { login: '<username>', seuser: '<selinux_user>', state: '<absent/present>'}

```
For example:
```sh
user_mapping:
  - { login: '__default__', seuser: 'unconfined_u', state: 'present' }
  - { login: 'niki', seuser: 'staff_u', state: 'present'}
```


### SELinux Booleans

For quick and simple SELinux configuration, parts of SELinux Policy are booleans. They can customize SELinux policy by enabling or disabling them. To turn on, change **state** to *on*, to turn off, change state to *off*. Each boolean has a description in the SELinux man page.
```
# Boolean description
      - { name: 'boolean_name', state: 'on/off', persistent: 'yes/no' }
```
For example:
```
# Allow cluster administrative domains to connect to the network using TCP.
      - { name: 'cluster_can_network_connect', state: 'off', persistent: 'yes' }
```
#### Memory protection
SELinux memory checks:
- execmem: make executable an anonymous mapping or private file mapping that is writable,
- execheap: make the heap executable,
- execstack: make the main process stack executable,
- execmod: make executable a file that has been modified by copy-on-write.

These memory checks allow for the blocking of known attacks, like buffer over-flows and code errors, that could lead to privilege escalations. If the process requiresone of the listed permissions, it is probably not good coding style, and should bereported to application maintainers.

SELinux has several booleans to prevent execution of memory, stack, and heap:

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| deny_execmem 		|Deny user domains applications to map a memory region as both executable and writable | on |
| 'domain'_execmem 		|Allow 'domain' to use executable memory and executable stack| off |
| selinuxuser_execmod   | Allow all unconfined executables to use libraries requiring text relocation that are not labeled textrel_shlib_t | off |
| selinuxuser_execheap  | Allow unconfined executables to make their heap memory executable. Doing this is a really bad idea. Probably indicates a badly coded executable, but could indicate an attack. This executable should be reported in bugzilla       |   off |
| selinuxuser_execstack | Allow unconfined executables to make their stack executable.  This should never, ever be necessary.Probably indicates a badly coded executable, but could indicate an attack. This executable should be reported in bugzilla       |    off |
| deny_ptrace	|Deny any process from ptracing or debugging any other processes | on |

The state of booleans for memory protection is configured in **/files/execmem.yml**. To allow domains execmem, certain booleans have to change value.


#### Prevent processes from sharing files

Files to share by Samba, GlusterFS or NFS are managed in */files/deny_export.yml*. If some files have to be shared, the related boolean needs to be turned on. Each boolean has a description.

##### Samba

Samba is using the file and printer sharing protocol. To share files via Samba, files have to be labeled as samba_share_t. In the SELinux policy for Samba booleans are defined , which allow sharing all files in home directories or all filesystem files or Network File System files 

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| smbd_anon_write		|Allow samba to modify public files used for public file transfer services. Files/Directories must be labeled public_content_rw_t | off |
| samba_enable_home_dirs |Allow samba to share users home directories| off |
| samba_export_all_rw |Allow samba to export all read/write | off |
| samba_export_all_ro |Allow samba to export all read only| off |
| samba_share_fusefs |Allow samba to export ntfs/fusefs volumes| off |
| samba_share_nfs| Allow samba to export NFS volumes | off |

##### Network File System

Network File System(NFS) is the protocol for sharing and storing files and directories, over the network. To share any read-only files and directories, or all files and directories via NFS, certain booleans have to be turned on.

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| nfsd_anon_write		| Allow nfs servers to modify public files used for public file transfer services. Files/Directories must be labeled public_content_rw_t | off |
| nfs_export_all_rw	| Allow nfs to export all read/write |  off |
| nfs_export_all_ro	|Allow nfs to export all read only |  off |


##### Gluster

GlusterFS is a scalable network filesystem suitable for data-intensive tasks such as cloud storage and media streaming Gluster services also require SELinux to be enabled, to share read-only files and directories, or all files and directories

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| gluster_anon_write		|Allow glusterfsd to modify public files used for public file transfer services. Files/Directories must be labeled public_content_rw_t | off |
| gluster_export_all_rw	| Allow glusterfsd to share any file/directory read-/write | off |
| gluster_export_all_ro |Allow glusterfsd to share any file/directory readonly | off |


#### Prevent processes from connecting to port
To prevent unauthorized connection or transfer of information, all booleans which allow domains to connect to any port will be disabled. Network connections of domains are determined in **./files/cant_connect.yml**. By default all domains have turned off booleans, which allow them to connect to network ports in this configuration. If some domain needs to connect to a port, the state of the relevant boolean has to be on. Each boolean has a description

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| mysql_connect_any		|Allow mysqld to connect to all ports | off |
| mysql_connect_http    | Allow mysqld to connect to http port | off |

## Run Ansile playbook

```
$ cd selinux-hardening
# ansible-playbook selinux-playbook.yml -i inventory/hosts
```


## Revert option

To return the SELinux configuration to before Ansible configuration, the revert option was added in **./revert/revert-playbook.yml**.Hosts are defined in the directory Inventory, in the file hosts.yaml, currently targeted on the localhost. When running this playbook, the user is asked if he wants to reboot the system. If yes, the system is rebooted afterward.


Run Revert playbook:
```
# ansible-playbook revert/revert-playbook.yml -i inventory/hosts
```



## System Lockdown

For system lockdown, there is another playbook, located in **system-lockdown/selinux-lockdown-playbook.yml**. In this playbook are used 3 powerful SELinux booleans, that disallow processes from transitioning to privileged user domains, prevent confined users from inserting Linux Kernel modules (e.g SELinux policymodule) and managing SELinux policy. Booleans are listed in the following table:

|  Boolean				| Description  | State |
| :------------ 		|:---------------:| -----:|
| secure_mode		|Do not allow transition to sysadm_t, sudo and su effected| on |
| secure_mode_insmod   | Do not allow any processes to load kernel modules  | on |
| secure_mode_policyload    | Do not allow any processes to modify kernel SELinux policy  | on |


When the last boolean, secure_mode_policyload, is enabled, users can onlydisable SELinux policy, during booting, by editing boot configuration. This playbook should be executed only when the user knows what he is doing. Hosts are again targeted on the localhost and can be defined in the directory Inventory in the file hosts.yaml. If this playbook is executed, with all secure booleans enabled, users will not beable to revert system lockdown, by executing the revert playbook. Moreover, userswill not be able to perform any operations with SELinux


Run system-lockdown playbook:
```
# ansible-playbook system-lockdown/selinux-lockdown-playbook.yml -i inventory/hosts
```

