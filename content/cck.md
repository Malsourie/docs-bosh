!!! Note
    Updated for bosh-release v183 (1.3010.0).

BOSH provides the `bosh cloud-check` CLI command (a.k.a `bosh cck`) to repair
IaaS resources used by a specific deployment. It is not commonly used during
normal operations; however, it becomes essential when some IaaS operations
fail and the Director cannot resolve problems without a human decision or when
the Resurrector is not enabled.

The Resurrector will only try to recover any VMs that are missing from the
IaaS or that have non-responsive agents. The `bosh cck` tool is similar to the
Resurrector in that it also looks for those two conditions; however, instead
of automatically trying to resolve these problems, it provides several options
to the operator.

In addition to looking for those two types of problems, `bosh cck` also checks
correct attachment and presence of persistent disks for each deployment job
instance.

Once the deployment is set via the `bosh deployment` command you can simply
run `bosh cck`. Here is an example output when no problems are detected:

```shell
bosh cck
```

```text
Performing cloud check...

Processing deployment manifest
------------------------------

Director task 622
  Started scanning 1 vms
  Started scanning 1 vms > Checking VM states. Done (00:00:00)
  Started scanning 1 vms > 1 OK, 0 unresponsive, 0 missing, 0 unbound, 0 out of sync. Done (00:00:00)
     Done scanning 1 vms (00:00:00)

  Started scanning 0 persistent disks
  Started scanning 0 persistent disks > Looking for inactive disks. Done (00:00:00)
  Started scanning 0 persistent disks > 0 OK, 0 missing, 0 inactive, 0 mount-info mismatch. Done (00:00:00)
     Done scanning 0 persistent disks (00:00:00)

Task 622 done

Started		2015-01-09 23:29:34 UTC
Finished	2015-01-09 23:29:34 UTC
Duration	00:00:00

Scan is complete, checking if any problems found...
No problems found
```

---
## Problems {: #problems }

### VM is missing {: #missing-vm }

Assuming there was a deployment with a VM, somehow that VM was deleted from
the IaaS outside of BOSH, here is what `cloud-check` would report:

```shell
bosh cck
```

```text
Performing cloud check...

Processing deployment manifest
------------------------------

Director task 623
  Started scanning 1 vms
  Started scanning 1 vms > Checking VM states. Done (00:00:10)
  Started scanning 1 vms > 0 OK, 0 unresponsive, 1 missing, 0 unbound, 0 out of sync. Done (00:00:00)
     Done scanning 1 vms (00:00:10)

  Started scanning 0 persistent disks
  Started scanning 0 persistent disks > Looking for inactive disks. Done (00:00:00)
  Started scanning 0 persistent disks > 0 OK, 0 missing, 0 inactive, 0 mount-info mismatch. Done (00:00:00)
     Done scanning 0 persistent disks (00:00:00)

Task 623 done

Started		2015-01-09 23:32:45 UTC
Finished	2015-01-09 23:32:56 UTC
Duration	00:00:11

Scan is complete, checking if any problems found...

Found 1 problem

Problem 1 of 1: VM with cloud ID `i-914c046a' missing.
  1. Skip for now
  2. Recreate VM
  3. Delete VM reference
Please choose a resolution [1 - 3]: 3

Below is the list of resolutions you've provided
Please make sure everything is fine and confirm your changes

  1. VM with cloud ID `i-914c046a' missing.
     Delete VM reference

Apply resolutions? (type 'yes' to continue): yes
Applying resolutions...

Director task 624
  Started applying problem resolutions > missing_vm 168: Delete VM reference. Done (00:00:00)

Task 624 done

Started		2015-01-09 23:33:20 UTC
Finished	2015-01-09 23:33:20 UTC
Duration	00:00:00
Cloudcheck is finished
```

`bosh cck` determined that the `i-914c046a` VM was missing. Possible options are:

1. `Skip for now`: the Director will not try to resolve this problem right now

2. `Recreate VM`: the Director will just create new VM, deploy specified
   releases according to deployed manifest and finally start release jobs

    Note on current behaviour: `cck` will not wait for all release job
    processes to start.

3. `Delete VM reference`: the Director will not create new VM in its place. If
   cck is run again, it will not report that VM is missing since it's not
   expected to exist. Running `bosh deploy` after cck will backfill missing
   VMs.

In the above example options 3 was picked and VM reference was deleted.

---
### VM is not responsive (unresponsive agent) {: #not-responsive-vm }

Assuming there was a deployment with a VM, somehow Agent is no longer
responding to the Director. In this situation `bosh vms` will report VM's
agent as `unresponsive agent`:

```shell
bosh -d simple-deployment vms --details
```

```text
Deployment `simple-deployment'

Director task 630

Task 630 done

+-----------------+--------------------+---------------+-----+------------+--------------------------------------+--------------+
| Job/index       | State              | Resource Pool | IPs | CID        | Agent ID                             | Resurrection |
+-----------------+--------------------+---------------+-----+------------+--------------------------------------+--------------+
| unknown/unknown | unresponsive agent |               |     | i-1db9ede6 | 59a30081-d63d-4c1b-80be-01fa681d8787 | active       |
+-----------------+--------------------+---------------+-----+------------+--------------------------------------+--------------+

VMs total: 1
```

Also `bosh deploy` will stop at `Binding existing deployment` stage since it is not able to communicate with unresponsive agent:

```shell
bosh deploy
```

```text
..snip...

Deploying
---------
Deployment name: `tiny-dummy.yml'
Director name: `idora'
Are you sure you want to deploy? (type 'yes' to continue): yes

Director task 631
  Started preparing deployment
  Started preparing deployment > Binding deployment. Done (00:00:00)
  Started preparing deployment > Binding releases. Done (00:00:00)
  Started preparing deployment > Binding existing deployment. Failed: Timed out sending `get_state' to 59a30081-d63d-4c1b-80be-01fa681d8787 after 45 seconds (00:02:15)

Error 450002: Timed out sending `get_state' to 59a30081-d63d-4c1b-80be-01fa681d8787 after 45 seconds

Task 631 error

For a more detailed error report, run: bosh task 631 --debug
```

```shell
bosh cck
```

```text
Performing cloud check...

Processing deployment manifest
------------------------------

Director task 640
  Started scanning 1 vms
  Started scanning 1 vms > Checking VM states. Done (00:00:10)
  Started scanning 1 vms > 0 OK, 1 unresponsive, 0 missing, 0 unbound, 0 out of sync. Done (00:00:00)
     Done scanning 1 vms (00:00:10)

  Started scanning 0 persistent disks
  Started scanning 0 persistent disks > Looking for inactive disks. Done (00:00:00)
  Started scanning 0 persistent disks > 0 OK, 0 missing, 0 inactive, 0 mount-info mismatch. Done (00:00:00)
     Done scanning 0 persistent disks (00:00:00)

Task 640 done

Started   2015-01-09 23:33:45 UTC
Finished  2015-01-09 23:33:55 UTC
Duration  00:00:10

Scan is complete, checking if any problems found...

Found 1 problem

Problem 1 of 1: dummy/0 (i-914c046a) is not responding.
  1. Skip for now
  2. Reboot VM
  3. Recreate VM
  4. Delete VM reference (forceful; may need to manually delete VM from the Cloud to avoid IP conflicts)
Please choose a resolution [1 - 4]: 4

Below is the list of resolutions you've provided
Please make sure everything is fine and confirm your changes

  1. dummy/0 (i-914c046a) is not responding.
     Delete VM reference (forceful; may need to manually delete VM from the Cloud to avoid IP conflicts)

Apply resolutions? (type 'yes' to continue): yes
Applying resolutions...

Director task 641
  Started applying problem resolutions > unresponsive_agent 168: Delete VM reference (...). Done (00:00:05)

Task 641 done

Started   2015-01-09 23:35:20 UTC
Finished  2015-01-09 23:35:25 UTC
Duration  00:00:05
Cloudcheck is finished
```

`cck` determined that `i-914c046a` VM is unresponsive. Possible options are:

1. `Skip for now`: the Director will not try to resolve this problem right now

2. `Reboot VM`: the Director will power off and then power on existing VM

    Note on current behaviour: `cck` will not wait for all release job
    processes to start.

3. `Recreate VM`: the Director will delete existing VM, then create a new VM,
   deploy specified releases according to deployed manifest and finally start
   release jobs

    Note on current behaviour: `cck` will not wait for all release job
    processes to start.

4. `Delete VM reference`: the Director will not create new VM in its place. If
   `cck` is run again, it will not report that VM is unresponsive since it does
   not exist. Running `bosh deploy` after cck will backfill missing VMs.

In the above example options 4 was picked and VM reference was deleted.

---

### Persistent Disk is not attached {: #unattached-persistent-disk }

Assuming there was a deployment with a VM, somehow persistent disk got detached.

```shell
bosh cck
```

```text
Performing cloud check...

Processing deployment manifest
------------------------------

Director task 656
  Started scanning 1 vms
  Started scanning 1 vms > Checking VM states. Done (00:00:00)
  Started scanning 1 vms > 1 OK, 0 unresponsive, 0 missing, 0 unbound, 0 out of sync. Done (00:00:00)
     Done scanning 1 vms (00:00:00)

  Started scanning 1 persistent disks
  Started scanning 1 persistent disks > Looking for inactive disks. Done (00:00:00)
  Started scanning 1 persistent disks > 0 OK, 0 missing, 0 inactive, 1 mount-info mismatch. Done (00:00:00)
     Done scanning 1 persistent disks (00:00:00)

Task 656 done

Started   2015-01-13 22:04:56 UTC
Finished  2015-01-13 22:04:56 UTC
Duration  00:00:00

Scan is complete, checking if any problems found...

Found 1 problem

Problem 1 of 1: Inconsistent mount information:
Record shows that disk 'vol-549f071f' should be mounted on i-4fcd99b4.
However it is currently :
  Not mounted in any VM.
  1. Skip for now
  2. Reattach disk to instance
  3. Reattach disk and reboot instance
Please choose a resolution [1 - 3]: 2

Below is the list of resolutions you've provided
Please make sure everything is fine and confirm your changes

  1. Inconsistent mount information:
Record shows that disk 'vol-549f071f' should be mounted on i-4fcd99b4.
However it is currently :
  Not mounted in any VM
     Reattach disk to instance

Apply resolutions? (type 'yes' to continue): yes
Applying resolutions...

Director task 657
  Started applying problem resolutions > mount_info_mismatch 23: Reattach disk to instance. Done (00:00:22)

Task 657 done

Started   2015-01-13 22:05:19 UTC
Finished  2015-01-13 22:05:41 UTC
Duration  00:00:22
Cloudcheck is finished
```

cck determined that `vol-549f071f` persistent disk is not attached to `i-4fcd99b4` VM. Possible options are:

1. `Skip for now`: the Director will not try to resolve this problem right now

2. `Reattach disk to instance`: the Director will reattach persistent disk to
   the VM and mount it at its usual location `/var/vcap/store`.

    Note on current behavior: Release job processes will not be restarted when
    persistent disk is remounted.

3. `Reattach disk and reboot instance`: the Director will reattach persistent
   disk to the VM and reboot it so that Agent can safely mount persistent disk
   before starting any release job processes.

    Note on current behavior: cck will not wait until VM reboots and restarts
    all release job processes.

---
### Inactive Disk {: #inactive-disk }

Assuming there are are disks that are marked as inactive,

```shell
bosh cck
```

```text
Using environment '10.0.0.5' as client 'admin'

Using deployment 'test-deployment'

Task 417

Task 417 | 23:59:42 | Scanning 7 VMs: Checking VM states (00:00:06)
Task 417 | 23:59:48 | Scanning 7 VMs: 7 OK, 0 unresponsive, 0 missing, 0 unbound (00:00:00)
Task 417 | 23:59:48 | Scanning 2 persistent disks: Looking for inactive disks (00:00:01)
Task 417 | 23:59:49 | Scanning 2 persistent disks: 0 OK, 0 missing, 2 inactive, 0 mount-info mismatch (00:00:00)

Task 417 Started  Tue Apr 21 23:59:42 UTC 2020
Task 417 Finished Tue Apr 21 23:59:49 UTC 2020
Task 417 Duration 00:00:07
Task 417 done

#   Type           Description
9   inactive_disk  Disk 'disk-eaca0b50-daf5-4fba-6dbf-06354e11e0af' (102400M) for instance 'database/5cc2ab64-a269-4c91-b6ba-9a6ff3c2d9d3 (0)' is inactive
10  inactive_disk  Disk 'disk-270c7959-5f06-4597-727c-6c4283afd138' (102400M) for instance 'blobstore/f048378e-27c8-4ab6-b08a-1c47d09570c5 (0)' is inactive

2 problems

1: Skip for now
2: Delete disk
3: Activate disk
Disk 'disk-eaca0b50-daf5-4fba-6dbf-06354e11e0af' (102400M) for instance 'database/5cc2ab64-a269-4c91-b6ba-9a6ff3c2d9d3 (0)' is inactive (1): 3

1: Skip for now
2: Delete disk
3: Activate disk
Disk 'disk-270c7959-5f06-4597-727c-6c4283afd138' (102400M) for instance 'blobstore/f048378e-27c8-4ab6-b08a-1c47d09570c5 (0)' is inactive (1): 3

Continue? [yN]: y


Task 419

Task 419 | 00:00:43 | Applying problem resolutions: Disk 'disk-eaca0b50-daf5-4fba-6dbf-06354e11e0af' (102400M) for instance 'database/5cc2ab64-a269-4c91-b6ba-9a6ff3c2d9d3 (0)' is inactive (inactive_disk 1): Activate disk (00:00:06)
Task 419 | 00:00:49 | Applying problem resolutions: Disk 'disk-270c7959-5f06-4597-727c-6c4283afd138' (102400M) for instance 'blobstore/f048378e-27c8-4ab6-b08a-1c47d09570c5 (0)' is inactive (inactive_disk 2): Activate disk (00:00:06)

Task 419 Started  Wed Apr 22 00:00:43 UTC 2020
Task 419 Finished Wed Apr 22 00:00:55 UTC 2020
Task 419 Duration 00:00:12
Task 419 done

Succeeded
```

cck determined that `disk-eaca0b50-daf5-4fba-6dbf-06354e11e0af` persistent disk is inactive. Possible options are:

1. `Skip for now`: the Director will not try to resolve this problem right now

2. `Delete disk`: the Director will check if the disk is unmounted, detach
   from any active VM, and mark it as an orphaned disk

3. `Activate disk`: the director will check if the disk is mounted and if the
   active VM already has a disk before marking the disk as active

---
### Persistent Disk is missing {: #missing-persistent-disk }

Assuming there was a deployment with a VM, somehow persistent disk got deleted.

Note: Not all CPIs implement needed functionality to determine if disk is missing. Those CPIs will report missing disk as [Persistent Disk is not attached](#unattached-persistent-disk) problem; however, both reattaching resolutions will fail since persistent disk would not be found.

---

## Automate recovery selection {: #automatic-recovery }

Automate recovery using the `--resolution=RESOLUTION-VALUE` cloud-check (cck)
option, where `RESOLUTION-VALUE` represents one of the following:

 * `ignore` - skip resolution
 * `recreate_vm` - recreate VM and wait for processes to start
 * `recreate_vm_without_wait` - recreate VM without waiting for processes to
   start (and thus [`post-start` scripts](job-lifecycle.md#start) won't run
   after processes have restarted, which could have consequences depending on
   how critical `post-start` scripts are)
 * `reboot_vm` - reboot the VM
 * `delete_vm` - delete the VM
 * `delete_vm_reference` - remove the VM reference that Director has (this
   could cause IaaS resources to be abandoned)
 * `delete_disk` - delete the disk
 * `delete_disk_reference` - remove the disk reference that Director has (this
   could cause IaaS resources to be abandoned)
 * `activate_disk` - mark the disk as active
 * `reattach_disk` - reattach persistent disk to the VM and mount it at its
   usual location `/var/vcap/store`
 * `reattach_disk_and_reboot`- reattach the persistent disk to the VM and
   reboot it so that Agent can safely mount persistent disk before starting
   any release job processes. cck will not wait until VM reboots and restarts
   all release job processes

!!! Warning
    Consider using `cck` interactively because the selected resolution will be
    applied to all problems that are found. Specifying `--resolution` can be
    risky if new, unexpected problems occur while you run the command and
    selected resolution may no longer be appropriate.
