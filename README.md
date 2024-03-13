### **Introduction**

Snapshotting is a feature of the Linux LVM. Snapshotting is a feature of the Linux LVM. All VMs which are installed on logical volumes on a Linux hypervisor can be "snapshotted" by using this feature. The snaphots includes the changes from the creation time of snapshot until now. If you need to rollback the snaphot, you reset the vm to the creation time (point-in-time-recovery).


### **Description of the role**

The name of the role is "lvmsnap".

It includes only one task file tasks/main.yml and one var file vars/main.yml.

All volumes are defined in the var file(roles/lvmsnap/vars/main.yml).

The section in the var file for a hosts starts with the hostname: and includes the volumes specification line by line.

Ther can only be one snapshot per vm using this playbook, this meening if you want a new snapshot you must first delete the old snapshot before taking a new.

hostname:

  `- {vg: VOLGROUPNAME, orgvol: LVNAME, snapsize: SIZE_OF_SNAPSHOTVOLUME(g)} `

Volgroupname = Name of the  volume group

orgvol = Name of the logical volumes, source for the snapshot

snapsize = Size of the snapshot in GB

The task file tasks/main.yml excutes a loop over the vars against the hostname.

```
olafs@stornesle:[lvmsnap]> tree

.

├── defaults

│   └── main.yml

├── handlers

│   └── main.yml

├── meta

│   └── main.yml

├── README.md

├── tasks

│   └── main.yml

├── tests

│   ├── inventory

│   └── test.yml

└── vars

    └── main.yml

```

### **Usage **

1.)

You need root rights on the hypervisor


2.)

checkout the git repo

git clone https://github.com/usegalaxy-no/lvm-snapshotting.git

3.)

change into the repo dir

cd lvm-snapshotting/

3.)

edit the playbook site-lvmsnap.yml

Add the hostname of the hypervisor and change the var "flavor" to predowntime to take a snapshot.

---
```
- hosts:

    - hestehov &lt;-- hostname of the hypervisor

  vars:

    flavour: predowntime &lt;-- to create snapshots

    #flavour: postdowntime  &lt;-- to delete snapshots

  become: true

  roles:

    - lvmsnap

```

4.) edit the var file roles/lvmsnap/var/main.yml

comment out only the line which you want to use

If a VM contains more than one disk, you MUST snapshot all the disks!

double check!

double double check!

```
---

# vars file for downtime

# usage:

# hostname:

#   - {vg: VOLGROUPNAME, orgvol: LVNAME, snapsize: SIZE_OF_SNAPSHOTVOLUME(g)}

hestehov:

    - { vg: vmguest_vg, orgvol:  db.usegalaxy.no-database, snapsize: 5g }

    - { vg: vmguest_vg, orgvol:  db.usegalaxy.no-root, snapsize: 5g }

    - { vg: vmguest_vg, orgvol:  db.usegalaxy.no-srv, snapsize: 50g }

    - { vg: vmguest_vg, orgvol:  slurm.usegalaxy.no-docker, snapsize: 5g }

    - { vg: vmguest_vg, orgvol:  slurm.usegalaxy.no-root, snapsize: 100g }

    - { vg: vmguest_vg, orgvol:  slurm.usegalaxy.no-tmp, snapsize: 10g }

    - { vg: vmguest_vg, orgvol:  usegalaxy.no-data, snapsize: 1000g }

    - { vg: vmguest_vg, orgvol:  usegalaxy.no-root, snapsize: 20g }

    - { vg: vmguest_vg, orgvol:  usegalaxy.no-srv, snapsize: 100g }
```


5.)

Run a dry-run
`ansible-playbook site-lvmsnap.yml --check -vvv`

and check the output

6.)

If everything is fine run the playbook
`ansible-playbook site-lvmsnap.yml`

7.) A good idea is to save a copy of the var file so you can delete all the snapshots easily after the maintenance action  using this var file

To the remove the snapshots, change the var "flavor" to postdowntime and run the playbook again. Use the same vars for the volnames etc.

What does the task file?

The task files runs a loop over the var file and creates or deletes the snapshots depending on the flavor var (predowntime|postdowntime).

The task file does not ask for confirmation while deleting, it deletes with force!

```
- name: Create a snapshot volume of the logical volume

  when: (flavour == 'predowntime')

  lvol:

    vg: '{{ item.vg }}'

    lv: '{{ item.orgvol }}'

    snapshot: 'SNAP{{ item.orgvol }}'

    size: '{{ item.snapsize }}'

  loop: '{{ vars[inventory_hostname] }}'

- name: Delete a snapshot volume of the logical volume

  when: (flavour == 'postdowntime')

  lvol:

    vg: '{{ item.vg }}'

    lv: 'SNAP{{ item.orgvol }}'

    state: absent

    force: yes

  loop: '{{ vars[inventory_hostname] }}'
```

### **Recover/reset a VM**

To recover a VM the VM must be powered off.

After that you can merge the snapshot into the original volume

`lvconvert --merge /path_to_vm_snapshot/`

(replace the device path with the path to your snapshot)
You can get the snapshot path by doing `lvdisplay`
(Remember you want to merge the snapshot)

If a VM contains more than one volume, you need to merge all the snapshots/volumes

For more information check:

[https://www.tecmint.com/take-snapshot-of-logical-volume-and-restore-in-lvm/](https://www.tecmint.com/take-snapshot-of-logical-volume-and-restore-in-lvm/)
