# Cinder Ceph Replication via Juju

## Environment Setup

Deploy the bundle via:
```
juju deploy ./ussuri-bionic-bundle.yaml
```

After the deployment is finished, initialize the Vault via:
```
./initialize-vault.sh
```

Export the necessary environment variables:
```
source ./openrc.sh
```

Create tenant networks:
```
openstack network create --external --provider-network-type flat --provider-physical-network external --share public
openstack subnet create --network public --subnet-range 10.114.0.0/16 --allocation-pool start=10.114.3.1,end=10.114.3.200 --gateway 10.114.1.1 --no-dhcp --ip-version 4 public_subnet

openstack router create public_router
openstack router set --external-gateway public public_router

openstack network create private
openstack subnet create --network private --subnet-range 10.33.22.0/24 --ip-version 4 --dns-nameserver 1.1.1.1 private_subnet
openstack router add subnet public_router private_subnet
```

Upload a Glance image to be used for testing:
```
curl -o /tmp/focal.img https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
openstack image create --public --container-format=bare --disk-format=qcow2 --file /tmp/focal.img focal
```

Create VMs flavor & keypair:
```
openstack flavor create --vcpus 1 --ram 2048 --disk 10 --ephemeral 10 m1.small
openstack keypair create mykey --public-key ~/.ssh/id_rsa.pub
```

Create the Cinder volumes type for replicated & non-replicated volumes in both sites:
```
openstack volume type create site-a-repl
openstack volume type set site-a-repl --property volume_backend_name=site-a-cinder-ceph
openstack volume type set site-a-repl --property replication_enabled='<is> True'

openstack volume type create site-a-local
openstack volume type set site-a-local --property volume_backend_name=site-a-cinder-ceph

openstack volume type create site-b-repl
openstack volume type set site-b-repl --property volume_backend_name=site-b-cinder-ceph
openstack volume type set site-b-repl --property replication_enabled='<is> True'

openstack volume type create site-b-local
openstack volume type set site-b-local --property volume_backend_name=site-b-cinder-ceph
```

## Cinder Ceph Replication scenarios

Before anything else, make sure your environment has the following [bug-fix](https://review.opendev.org/c/openstack/cinder/+/759315) applied.

### 1. Both sites are online, and we failover site-a with replicated & non-replicated non-attached Cinder volumes

Create the testing volumes:
```
openstack volume create --size 5 --type site-a-repl vol-site-a-replicated
openstack volume create --size 5 --type site-a-local vol-site-a-local
```
```
openstack volume list
+--------------------------------------+-----------------------+-----------+------+-------------+
| ID                                   | Name                  | Status    | Size | Attached to |
+--------------------------------------+-----------------------+-----------+------+-------------+
| 5e206014-bd5e-49aa-a847-8ebac7222f17 | vol-site-a-local      | available |    5 |             |
| 04b4c4e9-de5b-43ac-ae55-0898aeee87d8 | vol-site-a-replicated | available |    5 |             |
+--------------------------------------+-----------------------+-----------+------+-------------+
```

Execute the failover command:
```
cinder failover-host cinder@site-a-cinder-ceph
```

Wait until the failover is done:
```
cinder service-list
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
| Binary           | Host                                   | Zone | Status   | State | Updated_at                 | Cluster | Disabled Reason | Backend State |
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
...
| cinder-volume    | cinder@site-a-cinder-ceph              | nova | disabled | up    | 2021-01-13T16:38:06.000000 | -       | failed-over     | -             |
...
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
```
Notice the `cinder-volume` disabled with reason `failed-over` when everything is done.

Check the volumes again:
```
openstack volume list
+--------------------------------------+-----------------------+-----------+------+-------------+
| ID                                   | Name                  | Status    | Size | Attached to |
+--------------------------------------+-----------------------+-----------+------+-------------+
| 5e206014-bd5e-49aa-a847-8ebac7222f17 | vol-site-a-local      | error     |    5 |             |
| 04b4c4e9-de5b-43ac-ae55-0898aeee87d8 | vol-site-a-replicated | available |    5 |             |
+--------------------------------------+-----------------------+-----------+------+-------------+
```
The `vol-site-a-replicated` volume is available since it used the volume type
with replication enabled, and it survived the failover. However, the
`vol-site-a-local` with no replication, transitioned into error state. 

To failback, use the following command:
```
cinder failover-host cinder@site-a-cinder-ceph --backend_id default
```

After successful `failback` operation, the `cinder-volume` is not disabled anymore:
```
cinder service-list
+------------------+----------------------------------------+------+---------+-------+----------------------------+---------+-----------------+---------------+
| Binary           | Host                                   | Zone | Status  | State | Updated_at                 | Cluster | Disabled Reason | Backend State |
+------------------+----------------------------------------+------+---------+-------+----------------------------+---------+-----------------+---------------+
...
| cinder-volume    | cinder@site-a-cinder-ceph              | nova | enabled | up    | 2021-01-13T16:45:54.000000 | -       |                 | up            |
...
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
```

### 2. Both sites are online, and we failover site-a with replicated Cinder volumes used by VMs

Create a bootable volume, and start a new VM using it as the OS disk:
```
openstack volume create --size 5 --type site-a-repl --image focal --bootable focal-volume-site-a
```

Create another volume and attach it as secondary data volume to another VM:
```
openstack volume create --size 5 --type site-a-repl volume-site-a
```

Wait until both volumes are available to be used (it may take a while until the bootable volume is available):
```
openstack volume list
+--------------------------------------+---------------------+-----------+------+-------------+
| ID                                   | Name                | Status    | Size | Attached to |
+--------------------------------------+---------------------+-----------+------+-------------+
| 671b9a49-d402-40a9-8412-ca7a8d04f1cb | volume-site-a       | available |    5 |             |
| 049f545f-19c7-4f36-8b4b-2efc029c9e5a | focal-volume-site-a | available |    5 |             |
+--------------------------------------+---------------------+-----------+------+-------------+
```

Start the VMs:
```
openstack server create --flavor m1.small --key-name mykey --network private --image focal vm-site-a
openstack server create --flavor m1.small --key-name mykey --network private --volume focal-volume-site-a volume-vm-site-a
```

Wait until both of them are available:
```
openstack server list
+--------------------------------------+------------------+--------+----------------------+--------------------------+----------+
| ID                                   | Name             | Status | Networks             | Image                    | Flavor   |
+--------------------------------------+------------------+--------+----------------------+--------------------------+----------+
| 4ba2f7f1-e76f-4ece-be90-364b0d0939cd | volume-vm-site-a | ACTIVE | private=10.33.22.139 | N/A (booted from volume) | m1.small |
| 511ae520-be89-4759-aa24-67d8b18d6234 | vm-site-a        | ACTIVE | private=10.33.22.147 | focal                    | m1.small |
+--------------------------------------+------------------+--------+----------------------+--------------------------+----------+
```

Attach floating IPs, and the data volume to the VM booted from image:
```
VM_FIP=$(openstack floating ip create -f value -c floating_ip_address public)
openstack server add floating ip vm-site-a $VM_FIP
openstack server add volume vm-site-a volume-site-a
```
```
VOLUME_VM_FIP=$(openstack floating ip create -f value -c floating_ip_address public)
openstack server add floating ip volume-vm-site-a $VOLUME_VM_FIP
```

Write something on the attached volumes from both VMs (make sure you add the SSH security group rule):
```
ssh ubuntu@$VOLUME_VM_FIP
ubuntu@volume-vm-site-a:~$ echo "See you on the other side!" > data.txt
ubuntu@volume-vm-site-a:~$ sync
```
```
ssh ubuntu@$VM_FIP
ubuntu@vm-site-a:~$ sudo mkfs.ext4 /dev/vdc
ubuntu@vm-site-a:~$ mkdir data
ubuntu@vm-site-a:~$ sudo mount /dev/vdc ./data
ubuntu@vm-site-a:~$ sudo chown ubuntu.ubuntu ./data
ubuntu@vm-site-a:~$ echo "See you on the other side!" > data/data.txt
ubuntu@vm-site-a:~$ sync
```

We'll do the failover now.

#### IMPORTANT NOTES WITH CONTROLLED FAILOVER (both sites are online):

* It is **not** recommended to do the failover when Cinder volumes are `in-use`.

  During the failover, Cinder will try and demote Ceph images from the primary site, and if there is an active connection to it, the operation may fail, and the volume could transition to `error` state.

  Thus, we'll make sure the volumes are not `in-use` before doing the failover:
  ```
  ssh ubuntu@$VM_FIP sudo umount ./data
  openstack server remove volume vm-site-a volume-site-a
  openstack server stop volume-vm-site-a
  ```

* Make sure that the Ceph RBD Mirroring daemon finished its work (at least the initial replica sync), before doing the failover. Otherwise, the failover operation might not work (transitioning the volumes into `error` state). In this scenario, the cinder-volume service will log the following the error: `[errno 16] RBD image is busy (error promoting image)`.

  We can check volumes' mirroring state via the `status` Juju action (from the `ceph-rbd-mirror` charm). This needs to executed against the site-b `ceph-rbd-mirror` unit:
  ```
  $ echo 'verbose: True' > /tmp/status-params.yaml
  $ juju run-action --wait site-b-ceph-rbd-mirror/0 status --params /tmp/status-params.yaml
  unit-site-b-ceph-rbd-mirror-0:
    UnitId: site-b-ceph-rbd-mirror/0
    id: "69"
    results:
      output: |-
        cinder-ceph-a: health: WARNING
        daemon health: OK
        image health: WARNING
        images: 2 total
            2 unknown

        DAEMONS
        service 4633:
          instance_id: 5775
          client_id: juju-489814-default-27
          hostname: juju-489814-default-27
          version: 15.2.8
          leader: true
          health: OK


        IMAGES
        volume-663c7cb8-cd67-4a5f-b6a3-95e0a8af4acd:
          global_id:   f2842155-b3fc-40d2-920c-7bed4bd8fc3f
          state:       up+replaying
          description: replaying, {"bytes_per_second":17.5,"entries_behind_primary":0,"entries_per_second":0.2,"non_primary_position":{"entry_tid":3,"object_number":3,"tag_tid":1},"primary_position":{"entry_tid":3,"object_number":3,"tag_tid":1}}
          service:     juju-489814-default-27 on juju-489814-default-27
          last_update: 2021-04-12 15:11:17

        volume-cd920cfb-2d2c-441a-9664-41d49efef6f6:
          global_id:   6e96ab92-facd-440c-a330-44a1a6b5cb1b
          state:       up+replaying
          description: replaying, {"bytes_per_second":142336.67,"entries_behind_primary":3211,"entries_per_second":9.470000000000001,"non_primary_position":{"entry_tid":941,"object_number":1,"tag_tid":4},"primary_position":{"entry_tid":4152,"object_number":4,"tag_tid":4},"seconds_until_synced":339}
          service:     juju-489814-default-27 on juju-489814-default-27
          last_update: 2021-04-12 15:11:17
        cinder-ceph-b: health: OK
        daemon health: OK
        image health: OK
        images: 0 total

        DAEMONS
        service 4633:
          instance_id: 5818
          client_id: juju-489814-default-27
          hostname: juju-489814-default-27
          version: 15.2.8
          leader: true
          health: OK


        IMAGES
    status: completed
    timing:
      completed: 2021-04-12 15:11:42 +0000 UTC
      enqueued: 2021-04-12 15:11:39 +0000 UTC
      started: 2021-04-12 15:11:40 +0000 UTC
  ```

  The `IMAGES` section from the verbose `status` action output needs to be inspected.

  The secondary images that are already fully mirrored will report `"entries_behind_primary":0` (like `volume-663c7cb8-cd67-4a5f-b6a3-95e0a8af4acd` above). And the images that are still being mirrored will have `entries_behind_primary` bigger than `0` (like `"entries_behind_primary":3211` for `volume-cd920cfb-2d2c-441a-9664-41d49efef6f6` above).

  We need to have `"entries_behind_primary":0` given by all the secondary Ceph images.

Execute the Cinder failover:
```
cinder failover-host cinder@site-a-cinder-ceph
```

And wait until the failover is done:
```
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
| Binary           | Host                                   | Zone | Status   | State | Updated_at                 | Cluster | Disabled Reason | Backend State |
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
...
| cinder-volume    | cinder@site-a-cinder-ceph              | nova | disabled | up    | 2021-01-14T20:38:10.000000 | -       | failed-over     | -             |
...
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
```

Verify that data is up to date for the volume attached as secondary disk to the VM:
```
openstack server add volume vm-site-a volume-site-a

ssh ubuntu@$VM_FIP
ubuntu@vm-site-a:~$ sudo mount /dev/vdc ./data
ubuntu@vm-site-a:~$ cat ./data/data.txt
See you on the other side!
```

For the VM booted from a volume, we need to rebuild the VM, because Cinder needs to give the updated Ceph connection credentials to Nova:
```
openstack server delete volume-vm-site-a
openstack server create --flavor m1.small --key-name mykey --network private --volume focal-volume-site-a volume-vm-site-a
openstack server add floating ip volume-vm-site-a $VOLUME_VM_FIP

ssh ubuntu@$VOLUME_VM_FIP
ubuntu@volume-vm-site-a:~$ cat data.txt
See you on the other side!
```

### 3. Failover replicated Cinder volumes from offline site-a to online site-b

Basically, this is the DR (Disaster Recovery) scenario.

Create a testing volume:
```
openstack volume create --size 5 --type site-a-repl vol-site-a-replicated
```

Simulate a failure to the site-a Ceph cluster by shutting down the site-a Ceph monitor:
```
juju ssh site-a-ceph-mon/0 sudo poweroff
```

We'll do the failover now.

But before anything else, we need to temporarily adjust some timeouts to the Cinder Ceph backend. Without setting these, the failover takes an unreasonably amount of time to finish (or it may not even finish):
```
juju ssh site-a-cinder-ceph/0 sudo apt install crudini -y
juju ssh site-a-cinder-ceph/0 sudo crudini --set /etc/cinder/cinder.conf site-a-cinder-ceph rados_connect_timeout 1
juju ssh site-a-cinder-ceph/0 sudo crudini --set /etc/cinder/cinder.conf site-a-cinder-ceph rados_connection_retries 1
juju ssh site-a-cinder-ceph/0 sudo crudini --set /etc/cinder/cinder.conf site-a-cinder-ceph rados_connection_interval 0
juju ssh site-a-cinder-ceph/0 sudo crudini --set /etc/cinder/cinder.conf site-a-cinder-ceph replication_connect_timeout 1
juju ssh site-a-cinder-ceph/0 sudo systemctl restart cinder-volume
```

Execute the Cinder failover:
```
cinder failover-host cinder@site-a-cinder-ceph
```
and wait until it's done:
```
cinder service-list
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
| Binary           | Host                                   | Zone | Status   | State | Updated_at                 | Cluster | Disabled Reason | Backend State |
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
...
| cinder-volume    | cinder@site-a-cinder-ceph              | nova | disabled | up    | 2021-01-14T21:19:15.000000 | -       | failed-over     | -             |
...
+------------------+----------------------------------------+------+----------+-------+----------------------------+---------+-----------------+---------------+
```

Verify that the volume is available, and the cinder-volume log has the successful failover message:
```
openstack volume list
+--------------------------------------+-----------------------+-----------+------+-------------+
| ID                                   | Name                  | Status    | Size | Attached to |
+--------------------------------------+-----------------------+-----------+------+-------------+
| 4aa0db0d-cc8b-450a-afda-d06501cdd915 | vol-site-a-replicated | available |    5 |             |
+--------------------------------------+-----------------------+-----------+------+-------------+
```
```
...
2021-01-14 21:19:14.801 83270 INFO cinder.volume.drivers.rbd [req-f8913345-4518-4d96-a3d8-c2515deaf5a9 56b2814e6e9e4a1faf988ecd204ed547 87c75e65969f4e4980af8965f611a6d7 - be8b88fbb3944cdb931f6555c2a3771c be8b88fbb3944cdb931f6555c2a3771c] RBD driver failover completion started.
2021-01-14 21:19:14.801 83270 INFO cinder.volume.drivers.rbd [req-f8913345-4518-4d96-a3d8-c2515deaf5a9 56b2814e6e9e4a1faf988ecd204ed547 87c75e65969f4e4980af8965f611a6d7 - be8b88fbb3944cdb931f6555c2a3771c be8b88fbb3944cdb931f6555c2a3771c] RBD driver failover completion completed.
...
2021-01-14 21:19:14.818 83270 INFO cinder.volume.manager [req-f8913345-4518-4d96-a3d8-c2515deaf5a9 56b2814e6e9e4a1faf988ecd204ed547 87c75e65969f4e4980af8965f611a6d7 - be8b88fbb3944cdb931f6555c2a3771c be8b88fbb3944cdb931f6555c2a3771c] Failed over to replication target successfully.
...
```

Remove the timeouts from the Cinder Ceph backend, since these are not recommended for production use:
```
juju ssh site-a-cinder-ceph/0 sudo crudini --del /etc/cinder/cinder.conf site-a-cinder-ceph rados_connect_timeout
juju ssh site-a-cinder-ceph/0 sudo crudini --del /etc/cinder/cinder.conf site-a-cinder-ceph rados_connection_retries
juju ssh site-a-cinder-ceph/0 sudo crudini --del /etc/cinder/cinder.conf site-a-cinder-ceph rados_connection_interval
juju ssh site-a-cinder-ceph/0 sudo crudini --del /etc/cinder/cinder.conf site-a-cinder-ceph replication_connect_timeout
juju ssh site-a-cinder-ceph/0 sudo systemctl restart cinder-volume
```

The failover is finished right now, and we can use the volume as usual.

I will attach it to an existing VM, write some data on it, and detach it:
```
openstack server add volume vm-site-a vol-site-a-replicated

ssh ubuntu@$VM_FIP
ubuntu@vm-site-a:~$ sudo mkfs.ext4 /dev/vdc
ubuntu@vm-site-a:~$ mkdir data
ubuntu@vm-site-a:~$ sudo mount /dev/vdc ./data
ubuntu@vm-site-a:~$ sudo chown ubuntu.ubuntu ./data
ubuntu@vm-site-a:~$ echo "Some data!" > data/data.txt
ubuntu@vm-site-a:~$ sync
ubuntu@vm-site-a:~$ sudo umount ./data
ubuntu@vm-site-a:~$ exit

openstack server remove volume vm-site-a vol-site-a-replicated
```

### Data integrity before failback

When site-a is fixed, and it's online again, we will have two primary Ceph images for the testing Cinder volume, in site-a and site-b (split-brain scenario).

The Ceph RBD mirror will not sync the volume until the split-brain scenario is fixed manually.

There are two methods to solve the split-brain scenario:
1. Using Ceph `rbd` cli commands.
2. Using Juju actions.

The method `#2` is more user friendly, since the problem is solved via Juju commands only. So, it is the preferred method.

On the other hand, method `#1` needs: SSH-ing into the `ceph-rbd-mirror` unit, identifying each volume with replication enabled, and running the appropriate `rbd` cli call for it.

#### 1. Using Ceph `rbd` cli commands

Bring back site-a by starting the Ceph Mon there:
```
juju ssh 0

$ sudo lxc start juju-6fb684-0-lxd-5  # Identify the stopped LXD container via `sudo lxc list`
```

SSH into the site-a-rbd-mirror charm unit and switch to the root user:
```
juju ssh site-a-ceph-rbd-mirror/0
$ sudo su
```

Identify the RBD id:
```
ls /etc/ceph/
ceph.client.rbd-mirror.juju-6fb684-1-lxd-5.keyring  ceph.conf  rbdmap  remote.client.rbd-mirror.juju-6fb684-1-lxd-5.keyring  remote.conf
```

In the this example, the RBD id is `rbd-mirror.juju-6fb684-1-lxd-5`.

Demote the old site-a image:
```
rbd --id rbd-mirror.juju-6fb684-1-lxd-5 mirror image demote site-a-cinder-ceph/volume-4aa0db0d-cc8b-450a-afda-d06501cdd915
Image demoted to non-primary
```
where `4aa0db0d-cc8b-450a-afda-d06501cdd915` is Cinder volume id.

Re-sync the site-a image. This will bring the latest data from site-b back to site-a:
```
rbd --id rbd-mirror.juju-6fb684-1-lxd-5 mirror image resync site-a-cinder-ceph/volume-4aa0db0d-cc8b-450a-afda-d06501cdd915
Flagged image for resync from primary
```

As noted in the official documentation, and even in the output of the `image resync` command, the rbd command only flags the image to be resynced. And the Ceph RBD mirror daemon will be doing this in the background.

Wait until the resync is complete by fetching the mirror image status. We can query the image status via `image status` rbd command:
```
rbd --id rbd-mirror.juju-6fb684-1-lxd-5 mirror image status site-a-cinder-ceph/volume-4aa0db0d-cc8b-450a-afda-d06501cdd915
volume-4aa0db0d-cc8b-450a-afda-d06501cdd915:
  global_id:   3a4aa755-c9ee-4319-8ba4-fc494d20d783
  state:       up+syncing
  description: bootstrapping, IMAGE_SYNC/CREATE_SYNC_POINT
  service:     juju-6fb684-1-lxd-5 on juju-6fb684-1-lxd-5
  last_update: 2021-01-14 22:20:49
```

As long as the state is still `up+syncing`, we have to wait for the resync to be complete. When the resync is complete, the state will be `up+replaying`:
```
rbd --id rbd-mirror.juju-6fb684-1-lxd-5 mirror image status site-a-cinder-ceph/volume-4aa0db0d-cc8b-450a-afda-d06501cdd915
volume-4aa0db0d-cc8b-450a-afda-d06501cdd915:
  global_id:   3a4aa755-c9ee-4319-8ba4-fc494d20d783
  state:       up+replaying
  description: replaying, {"bytes_per_second":3805440.87,"entries_behind_primary":2,"entries_per_second":234.8,"non_primary_position":{"entry_tid":1,"object_number":9,"tag_tid":4},"primary_position":{"entry_tid":3,"object_number":11,"tag_tid":4},"seconds_until_synced":0}
  service:     juju-6fb684-1-lxd-5 on juju-6fb684-1-lxd-5
  last_update: 2021-01-14 22:21:49
```

If there are more replicated volumes, you need to do the same procedure for all of them.

#### 2. Using Juju actions

Demote the Ceph images from site-a using:
```
juju run-action --wait site-a-ceph-rbd-mirror/0 demote --string-args pools="site-a-cinder-ceph"
```

Re-sync the site-a images via:
```
juju run-action --wait site-a-ceph-rbd-mirror/0 resync-pools --string-args pools="site-a-cinder-ceph"
```

Keep in mind that this only flags the site-a images to be re-synced against the primary images from site-b, and the Ceph RBD mirror daemon does this in the background.

We need to wait until the operation is complete for all the volumes. The operation can be monitored using the `status` Juju action with `verbose` flag:
```
echo 'verbose: True' > /tmp/status-params.yaml
juju run-action --wait site-a-ceph-rbd-mirror/0 status --params /tmp/status-params.yaml
```

The `IMAGES` section from the action output needs to be inspected.

As long as there are images reporting state `up+syncing`, the re-sync operation is not complete. Keep in mind that only images marked with replication enabled are re-synced.

When the resync operation is complete, all the replicated volumes will report `state: up+replaying` and `"entries_behind_primary":0`. We are ready to failback at this point.

#### Failback

Once all the volumes are properly demoted + resynced, we can do the actual failback via the `cinder` cli:
```
cinder failover-host cinder@site-a-cinder-ceph --backend_id default
```

When the failback operation ends, re-attach the cinder volume to the VM, SSH into it and check data from it:
```
openstack server add volume vm-site-a vol-site-a-replicated

ssh ubuntu@$VM_FIP
ubuntu@vm-site-a:~$ sudo mount /dev/vdc ./data
ubuntu@vm-site-a:~$ ls -l ./data/
total 20
-rw-rw-r-- 1 ubuntu ubuntu    11 Jan 14 21:31 data.txt
drwx------ 2 root   root   16384 Jan 14 21:30 lost+found
ubuntu@vm-site-a:~$ cat ./data/data.txt
Some data!
```

Everything went smooth, and we properly did the failback, after recovering site-a.

Just as a double check, we can go back to the LXD container with rbd-mirror and validate that the site-a image is primary now:
```
juju ssh site-a-ceph-rbd-mirror/0

sudo rbd --id rbd-mirror.juju-6fb684-1-lxd-5 mirror image status site-a-cinder-ceph/volume-4aa0db0d-cc8b-450a-afda-d06501cdd915
volume-4aa0db0d-cc8b-450a-afda-d06501cdd915:
  global_id:   3a4aa755-c9ee-4319-8ba4-fc494d20d783
  state:       up+stopped
  description: local image is primary
  service:     juju-6fb684-1-lxd-5 on juju-6fb684-1-lxd-5
  last_update: 2021-01-14 22:28:19
```

Or you can use the Juju action to do that:
```
echo 'verbose: True' > /tmp/status-params.yaml
juju run-action --wait site-a-ceph-rbd-mirror/0 status --params /tmp/status-params.yaml
```

You may notice images with `state: up+stopped` and `description: local image is primary`.
