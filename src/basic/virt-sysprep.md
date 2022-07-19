
# virt-sysprep - 初始化虚拟机副本工具

## 背景
为了能够在模拟环境中快速创建KVM虚拟机，需要以虚拟机作为模版，快速clone出需要的部署集群所需虚拟机。

## virt-sysprep 是什么
virt-clone命令可以复制一个已经存在的虚拟机，这个命令只能在vm停机状态使用，它将克隆已存在VM的所有信息，包括UUID和MAC地址。

可以使用virt-sysprep工具来配置新克隆的VM。virt-sysperp初始化虚拟机实例。virt-sysperp会将虚假机初始化到系统刚安装的状态，它会删除掉虚拟机中的ssh key文件、重置网络MAC地址、主机名以及系统用户。

# install
```bash
yum whatprovides */virt-sysprep
yum install libguestfs-tools -y
```

# run
```bash
#初始化
[root@kvm-node1 images]# virt-sysprep -d kvm-clone1
[ 0.0] Examining the guest ...
[ 44.9] Performing "abrt-data" ...
[ 44.9] Performing "bash-history" ...
[ 44.9] Performing "blkid-tab" ...
[ 44.9] Performing "crash-data" ...
[ 44.9] Performing "cron-spool" ...
[ 44.9] Performing "dhcp-client-state" ...
[ 44.9] Performing "dhcp-server-state" ...
[ 44.9] Performing "dovecot-data" ...
[ 44.9] Performing "logfiles" ...
[ 44.9] Performing "machine-id" ...
[ 44.9] Performing "mail-spool" ...
[ 44.9] Performing "net-hostname" ...
[ 44.9] Performing "net-hwaddr" ...
[ 44.9] Performing "pacct-log" ...
[ 44.9] Performing "package-manager-cache" ...
[ 44.9] Performing "pam-data" ...
[ 44.9] Performing "puppet-data-log" ...
[ 44.9] Performing "rh-subscription-manager" ...
[ 44.9] Performing "rhn-systemid" ...
[ 44.9] Performing "rpm-db" ...
[ 44.9] Performing "samba-db-log" ...
[ 44.9] Performing "script" ...
[ 44.9] Performing "smolt-uuid" ...
[ 44.9] Performing "ssh-hostkeys" ...
[ 44.9] Performing "ssh-userdir" ...
[ 44.9] Performing "sssd-db-log" ...
[ 44.9] Performing "tmp-files" ...
[ 45.0] Performing "udev-persistent-net" ...
[ 45.0] Performing "utmp" ...
[ 45.0] Performing "yum-uuid" ...
[ 45.0] Performing "customize" ...
[ 45.0] Setting a random seed
[ 45.0] Performing "lvm-uuids" ...
```

```bash
# virt-sysprep参数很多，能配置的地方也很多，举个常用的配置hostname和root密码的例子：重置虚拟机主机名和root用户账号(这里密码案例是 CHANGE_ME)
[root@kvm-node1 images]# virt-sysprep -d devstack --hostname devstack --root-password password:CHANGE_ME
```

```bash
[root@kvm-node1 images]# virt-sysprep --list-operations
abrt-data * Remove the crash data generated by ABRT
bash-history * Remove the bash history in the guest
blkid-tab * Remove blkid tab in the guest
ca-certificates   Remove CA certificates in the guest
crash-data * Remove the crash data generated by kexec-tools
cron-spool * Remove user at-jobs and cron-jobs
customize * Customize the guest
dhcp-client-state * Remove DHCP client leases
dhcp-server-state * Remove DHCP server leases
dovecot-data * Remove Dovecot (mail server) data
firewall-rules   Remove the firewall rules
flag-reconfiguration   Flag the system for reconfiguration
fs-uuids   Change filesystem UUIDs
kerberos-data   Remove Kerberos data in the guest
logfiles * Remove many log files from the guest
lvm-uuids * Change LVM2 PV and VG UUIDs
machine-id * Remove the local machine ID
mail-spool * Remove email from the local mail spool directory
net-hostname * Remove HOSTNAME in network interface configuration
net-hwaddr * Remove HWADDR (hard-coded MAC address) configuration
pacct-log * Remove the process accounting log files
package-manager-cache * Remove package manager cache
pam-data * Remove the PAM data in the guest
puppet-data-log * Remove the data and log files of puppet
rh-subscription-manager * Remove the RH subscription manager files
rhn-systemid * Remove the RHN system ID
rpm-db * Remove host-specific RPM database files
samba-db-log * Remove the database and log files of Samba
script * Run arbitrary scripts against the guest
smolt-uuid * Remove the Smolt hardware UUID
ssh-hostkeys * Remove the SSH host keys in the guest
ssh-userdir * Remove ".ssh" directories in the guest
sssd-db-log * Remove the database and log files of sssd
tmp-files * Remove temporary files
udev-persistent-net * Remove udev persistent net rules
user-account   Remove the user accounts in the guest
utmp * Remove the utmp file
yum-uuid * Remove the yum UUID
```

