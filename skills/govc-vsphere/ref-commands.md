# govc Command Reference

Full flag tables and examples for govc commands. See the main [SKILL.md](SKILL.md) for
core workflows and common patterns.

---

## vm.create

```
Usage: govc vm.create [OPTIONS] NAME

Options:
  -annotation=           VM description
  -c=1                   Number of CPUs
  -cluster=              Use cluster for VM placement via DRS
  -datastore-cluster=    Datastore cluster [GOVC_DATASTORE_CLUSTER]
  -disk=                 Disk path (existing) OR size (new, e.g. 20GB)
  -disk-datastore=       Datastore for disk file
  -disk.controller=scsi  Disk controller type
  -disk.eager=false      Eagerly scrub new disk
  -disk.thick=false      Thick provision new disk
  -ds=                   Datastore [GOVC_DATASTORE]
  -firmware=bios         Firmware type [bios|efi]
  -folder=               Inventory folder [GOVC_FOLDER]
  -force=false           Create VM if vmx already exists
  -g=otherGuest          Guest OS ID
  -host=                 Host system [GOVC_HOST]
  -iso=                  ISO path
  -iso-datastore=        Datastore for ISO file
  -link=true             Link specified disk
  -m=1024                Size in MB of memory
  -net=                  Network [GOVC_NETWORK]
  -net.adapter=e1000     Network adapter type
  -on=true               Power on VM
  -pool=                 Resource pool [GOVC_RESOURCE_POOL]
  -profile=[]            Storage profile name or ID
  -version=              ESXi hardware version
```

Guest OS IDs: use `govc vm.option.info` or `govc vm.option.ls` to discover valid `-g` values.

Examples:

```bash
govc vm.create -on=false vm-name
govc vm.create -m 2048 -c 2 -g freebsd64Guest -net.adapter vmxnet3 -disk.controller pvscsi vm-name
govc vm.create -cluster cluster1 vm-name
govc vm.create -iso library:/boot/linux/ubuntu.iso vm-name
govc vm.create -m 4096 -c 4 -g ubuntu64Guest -firmware efi -disk 40GB -ds datastore1 my-vm
```

---

## vm.clone

```
Usage: govc vm.clone [OPTIONS] NAME

Options:
  -annotation=           VM description
  -c=0                   Number of CPUs
  -cluster=              Use cluster for VM placement via DRS
  -customization=        Customization Specification Name
  -datastore-cluster=    Datastore cluster [GOVC_DATASTORE_CLUSTER]
  -ds=                   Datastore [GOVC_DATASTORE]
  -folder=               Inventory folder [GOVC_FOLDER]
  -force=false           Create VM if vmx already exists
  -host=                 Host system [GOVC_HOST]
  -link=false            Creates a linked clone from snapshot or source VM
  -m=0                   Size in MB of memory
  -net=                  Network [GOVC_NETWORK]
  -net.adapter=e1000     Network adapter type
  -on=true               Power on VM
  -pool=                 Resource pool [GOVC_RESOURCE_POOL]
  -snapshot=             Snapshot name to clone from
  -template=false        Create a Template
  -vm=                   Source virtual machine [GOVC_VM]
  -waitip=false          Wait for VM to acquire IP address
```

Examples:

```bash
govc vm.clone -vm template-vm new-vm
govc vm.clone -vm template-vm -link new-vm
govc vm.clone -vm template-vm -snapshot s-name new-vm
govc vm.clone -vm template-vm -link -snapshot s-name new-vm
govc vm.clone -vm template-vm -cluster cluster1 new-vm
govc vm.clone -vm template-vm -template new-template
govc vm.clone -vm=/DC/vm/Folder/template -on=true -host=esxi01 -ds=datastore01 myVM
```

---

## vm.change

```
Usage: govc vm.change [OPTIONS]

Options:
  -annotation=                   VM description
  -c=0                           Number of CPUs
  -cpu-hot-add-enabled=<nil>     Enable CPU hot add
  -cpu.limit=<nil>               CPU limit in MHz
  -cpu.reservation=<nil>         CPU reservation in MHz
  -cpu.shares=                   CPU shares level or number
  -e=[]                          ExtraConfig. <key>=<value>
  -f=[]                          ExtraConfig. <key>=<absolute path to file>
  -g=                            Guest OS
  -latency=                      Latency sensitivity (low|normal|medium|high)
  -m=0                           Size in MB of memory
  -mem.limit=<nil>               Memory limit in MB
  -mem.reservation=<nil>         Memory reservation in MB
  -mem.shares=                   Memory shares level or number
  -memory-hot-add-enabled=<nil>  Enable memory hot add
  -memory-pin=<nil>              Reserve all guest memory
  -name=                         Display name
  -nested-hv-enabled=<nil>       Enable nested hardware-assisted virtualization
  -sync-time-with-host=<nil>     Enable SyncTimeWithHost
  -uuid=                         BIOS UUID
  -vm=                           Virtual machine [GOVC_VM]
```

Examples:

```bash
govc vm.change -vm my-vm -m 4096
govc vm.change -vm my-vm -c 4
govc vm.change -vm my-vm -cpu-hot-add-enabled -memory-hot-add-enabled
govc vm.change -vm my-vm -e smc.present=TRUE -e ich7m.present=TRUE
govc vm.change -vm my-vm -e guestinfo.vmname my-vm
govc vm.change -vm my-vm -latency high
govc vm.change -vm my-vm -nested-hv-enabled=true
```

---

## vm.customize

```
Usage: govc vm.customize [OPTIONS] [NAME]

Customize VM. Optionally specify a customization spec NAME.
```

Examples:

```bash
govc vm.customize -vm my-vm -name my-hostname -ip dhcp
govc vm.customize -vm my-vm -gateway 10.0.0.1 -ip 10.0.0.100 -netmask 255.255.255.0 -dns-server 8.8.8.8 spec-name
govc vm.customize -vm my-vm -ip 10.0.0.178 -netmask 255.255.255.0 -ip 10.0.0.162 -netmask 255.255.255.0
```

---

## vm.power

```
Usage: govc vm.power [OPTIONS] VM...

Options:
  -M=false     Use VirtualMachine.Migrate
  -force=false Force (ignore guest OS)
  -off=false   Power off
  -on=false    Power on
  -r=false     Reset
  -s=false     Shutdown guest (requires VMware Tools)
  -suspend=false  Suspend
```

Examples:

```bash
govc vm.power -on=true my-vm
govc vm.power -s=true my-vm       # graceful shutdown
govc vm.power -off=true my-vm     # force off
govc vm.power -r=true my-vm       # reset
govc vm.power -suspend=true my-vm
```

---

## vm.destroy

```
Usage: govc vm.destroy [OPTIONS] VM...

Power off and delete VM. Attached virtual disks are also deleted.
Use 'device.remove -vm VM -keep disk-*' to detach disks before destroying.
```

---

## vm.info

```
Usage: govc vm.info [OPTIONS] VM...

Options:
  -e=false       Show ExtraConfig
  -g=true        Show general summary
  -r=false       Show resource summary
  -t=false       Show ToolsConfigInfo
  -waitip=false  Wait for VM to acquire IP address
```

Examples:

```bash
govc vm.info my-vm
govc vm.info -r my-vm
govc vm.info -json my-vm
govc find . -type m -runtime.powerState poweredOn | xargs govc vm.info
```

---

## vm.ip

```
Usage: govc vm.ip [OPTIONS] VM...

Options:
  -a=false    Report all IPs (one per NIC, comma delimited)
  -n=false    Report NICs and IPs
  -v4=false   Only report IPv4 addresses
  -wait=0s    Wait time for IP (0 = indefinitely until IP is reported)
```

Examples:

```bash
govc vm.ip my-vm
govc vm.ip -a my-vm
govc vm.ip -v4 -a my-vm
```

---

## vm.migrate

```
Usage: govc vm.migrate [OPTIONS] VM...

Migrates VM to a specific resource pool, host, or both.

Options:
  -ds=         Datastore [GOVC_DATASTORE]
  -host=       Host system [GOVC_HOST]
  -net=        Network [GOVC_NETWORK]
  -pool=       Resource pool [GOVC_RESOURCE_POOL]
  -priority=defaultPriority  Migration priority (defaultPriority|highPriority|lowPriority)
```

Examples:

```bash
govc vm.migrate -host esxi02 my-vm
govc vm.migrate -pool /dc/host/cluster/Resources -ds datastore2 my-vm
```

---

## snapshot.create

```
Usage: govc snapshot.create [OPTIONS] NAME

Options:
  -d=       Snapshot description
  -m=true   Include memory state
  -q=false  Quiesce guest file system
  -vm=      Virtual machine [GOVC_VM]
```

Examples:

```bash
govc snapshot.create -vm my-vm before-update
govc snapshot.create -vm my-vm -d "Pre-patch snapshot" -q=true pre-patch
```

---

## snapshot.tree

```
Usage: govc snapshot.tree [OPTIONS]

Options:
  -C=false  Print the current snapshot name only
  -D=false  Print the snapshot creation date
  -c=true   Print the current snapshot
  -d=false  Print the snapshot description
  -f=false  Print the full path prefix
  -i=false  Print the snapshot id
  -s=false  Print the snapshot size
  -vm=      Virtual machine [GOVC_VM]
```

Examples:

```bash
govc snapshot.tree -vm my-vm
govc snapshot.tree -vm my-vm -D -i -d
govc snapshot.tree -vm my-vm -C     # current snapshot name only
```

---

## snapshot.revert

```
Usage: govc snapshot.revert [OPTIONS] [NAME]

Revert to snapshot. If NAME is not provided, revert to the current snapshot.
NAME can be the snapshot name, tree path, or moid.

Options:
  -s=false  Suppress power on
  -vm=      Virtual machine [GOVC_VM]
```

---

## snapshot.remove

```
Usage: govc snapshot.remove [OPTIONS] NAME

NAME can be the snapshot name, tree path, moid, or '*' to remove all.

Options:
  -c=true   Consolidate disks
  -r=false  Remove snapshot children
  -vm=      Virtual machine [GOVC_VM]
```

---

## snapshot.export

```
Usage: govc snapshot.export [OPTIONS] NAME

Export snapshot of VM. NAME can be snapshot name, tree path, or moid.

Options:
  -d=.       Destination directory
  -lease=false  Output NFC Lease only
  -vm=       Virtual machine [GOVC_VM]
```

---

## vm.disk.create

```
Usage: govc vm.disk.create [OPTIONS]

Options:
  -controller=           Disk controller
  -ds=                   Datastore [GOVC_DATASTORE]
  -eager=false           Eagerly scrub new disk
  -mode=persistent       Disk mode
  -name=                 Name for new disk
  -sharing=              Sharing (sharingNone|sharingMultiWriter)
  -size=10.0GB           Size of new disk
  -thick=false           Thick provision new disk
  -vm=                   Virtual machine [GOVC_VM]
```

Examples:

```bash
govc vm.disk.create -vm my-vm -name my-vm/disk1 -size 10G
govc vm.disk.create -vm my-vm -name my-vm/disk2 -size 50G -eager -thick
```

---

## vm.disk.attach

```
Usage: govc vm.disk.attach [OPTIONS]

Options:
  -controller=       Disk controller
  -disk=             Disk path name
  -ds=               Datastore [GOVC_DATASTORE]
  -link=true         Link specified disk
  -mode=             Disk mode
  -persist=true      Persist attached disk
  -sharing=          Sharing (sharingNone|sharingMultiWriter)
  -vm=               Virtual machine [GOVC_VM]
```

Examples:

```bash
govc vm.disk.attach -vm my-vm -disk my-vm/disk1.vmdk
govc vm.disk.attach -vm my-vm -disk shared/data.vmdk -link=false -sharing sharingMultiWriter
```

---

## vm.disk.change

```
Usage: govc vm.disk.change [OPTIONS]

Options:
  -disk.filePath=    Disk file name
  -disk.key=0        Disk unique key
  -disk.label=       Disk label
  -disk.name=        Disk name
  -mode=             Disk mode
  -sharing=          Sharing
  -size=0B           New disk size
  -vm=               Virtual machine [GOVC_VM]
```

Examples:

```bash
govc vm.disk.change -vm my-vm -disk.key 2001 -size 100G
govc vm.disk.change -vm my-vm -disk.label "Hard disk 2" -size 50G
```

---

## datastore.info

```
Usage: govc datastore.info [OPTIONS] [PATH]...

Display datastore capacity and usage.
```

---

## datastore.ls

```
Usage: govc datastore.ls [OPTIONS] [PATH]...

Options:
  -R=false   Recursive listing
  -a=false   Show hidden files
  -l=false   Long listing format
  -p=false   Append / to directories
```

---

## datastore.upload / datastore.download

```
Usage: govc datastore.upload [OPTIONS] LOCAL REMOTE
Usage: govc datastore.download [OPTIONS] REMOTE LOCAL
```

Examples:

```bash
govc datastore.upload -ds datastore1 local.iso remote/path.iso
govc datastore.download -ds datastore1 remote/path.iso local.iso
```

---

## import.ova / import.ovf

```
Usage: govc import.ova [OPTIONS] PATH_TO_OVA
Usage: govc import.ovf [OPTIONS] PATH_TO_OVF

Options:
  -ds=       Datastore [GOVC_DATASTORE]
  -folder=   Inventory folder [GOVC_FOLDER]
  -host=     Host system [GOVC_HOST]
  -name=     Name to use for new entity
  -options=  Options JSON file or stdin (-)
  -pool=     Resource pool [GOVC_RESOURCE_POOL]
```

Examples:

```bash
govc import.ova my-appliance.ova
govc import.ova -folder=templates -name=my-template my-appliance.ova
govc import.ova -options - my-appliance.ova <<EOF
{"DiskProvisioning": "thin", "MarkAsTemplate": true}
EOF
```

---

## export.ovf

```
Usage: govc export.ovf [OPTIONS]

Options:
  -f=false   Overwrite existing
  -i=false   Include image files (*.nvram)
  -name=     Destination directory name
  -prefix=   Filename prefix
  -vm=       Virtual machine [GOVC_VM]
```

Examples:

```bash
govc export.ovf -vm my-vm -f=true .
govc export.ovf -vm my-vm -f=true -i=true ./exports/
```

---

## vm.markastemplate / vm.markasvm

```
Usage: govc vm.markastemplate [OPTIONS] VM
Usage: govc vm.markasvm [OPTIONS] VM

Options for markasvm:
  -host=  Host system [GOVC_HOST]
  -pool=  Resource pool [GOVC_RESOURCE_POOL]
```

Examples:

```bash
govc vm.markastemplate /mydc/vm/my-vm
esxhost=$(govc vm.info /mydc/vm/my-template | grep 'Host:' | awk '{print $2}')
govc vm.markasvm -host=$esxhost /mydc/vm/my-template
```

---

## find

```
Usage: govc find [OPTIONS] [ROOT] [KEY VAL]...

Options:
  -i=false           Print managed object reference
  -maxdepth=-1       Max depth
  -name=             Name pattern (glob)
  -type=[]           Resource type (d=Datacenter, c=ClusterComputeResource,
                     h=HostSystem, m=VirtualMachine, n=Network, s=Datastore,
                     p=DistributedVirtualPortgroup, r=ResourcePool, f=Folder)
```

Examples:

```bash
govc find / -type m                              # all VMs
govc find / -type m -name 'prod-*'               # VMs by name
govc find / -type m -runtime.powerState poweredOn # powered-on VMs
govc find / -type h                              # all hosts
govc find / -type s                              # all datastores
govc find / -type n                              # all networks
```

---

## object.collect

```
Usage: govc object.collect [OPTIONS] MANAGED_OBJECT [PROPERTY]...

Retrieve managed object properties.
```

Examples:

```bash
govc object.collect /mydc/vm/my-vm
govc object.collect -json /mydc/vm/my-vm guest.ipAddress
```

---

## guest.run / guest.upload / guest.download

Requires VMware Tools running in the guest. Credentials are passed via `-l user:password`.

```bash
govc guest.run -vm my-vm -l user:pass /bin/uname -a
govc guest.upload -vm my-vm -l user:pass local.conf /etc/app.conf
govc guest.download -vm my-vm -l user:pass /var/log/app.log ./app.log
govc guest.ls -vm my-vm -l user:pass /tmp/
govc guest.mkdir -vm my-vm -l user:pass /opt/myapp
```

---

## host.info

```
Usage: govc host.info [OPTIONS] HOST...
```

Examples:

```bash
govc host.info /mydc/host/cluster/*
govc host.info -json /mydc/host/cluster/* | jq '.hostSystems[].summary.config.product.version'
```

---

## Useful Scripting Patterns

### Get ESXi versions for all hosts in a cluster

```bash
dc=mydc; cluster=mycluster
for host in $(govc ls /$dc/host/$cluster | grep -v Resources); do
  govc host.info -json $host | jq -r '.hostSystems[].summary.config.product.version'
done
```

### Bulk power off VMs matching a pattern

```bash
govc find / -type m -name 'test-*' -runtime.powerState poweredOn | xargs -I{} govc vm.power -s=true {}
```

### List all VMs with their IPs

```bash
govc find / -type m | while read vm; do
  ip=$(govc vm.ip -wait 1s "$vm" 2>/dev/null || echo "N/A")
  echo "$vm -> $ip"
done
```
