# Nimble Deployment Guide for RHOSP16.2

## Overview

This page provides steps to deploy & configure Nimble backend driver for RHOSP16.2.

## Prerequisites

* Red Hat OpenStack Platform 16.2 with RHEL 8.4.

* Nimble array 6.0.0 or higher.

## Steps

### 1.  Prepare the Environment File for  nimble backend

#### 1.1 Environment File for nimble backend

The environment file is an OSP director environment file. The environment file contains the settings for each backend you want to define.

Create the environment file “cinder-nimble-iscsi.yaml” under /home/stack/templates/ with below parameters and other backend details.

```
parameter_defaults:
  CinderEnableIscsiBackend: false
  ControllerExtraConfig:
```

Sample files for iSCSI backend is available in [templates](https://github.com/hpe-storage/hpe-nimble-cinder-rhosp16.2/blob/master/templates) folder for reference.

#### Additional Help

For further details of Nimble cinder driver, kindly refer documentation [here](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/nimble-volume-driver.html)


### 2.  Deploy the overcloud and configured backends

After creating ```cinder-nimble-iscsi.yaml``` file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option.
Use the ```-e``` option to include the environment file ```cinder-nimble-iscsi.yaml```.

The order of the environment files (.yaml) is important as the parameters and resources defined in subsequent environment files take precedence.

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates \
    -e /home/stack/templates/node-info.yaml \
    -e /home/stack/containers-prepare-parameter.yaml \
    -e /home/stack/templates/cinder-nimble-iscsi.yaml \
```
### 3.  Verify the configured changes

3.1 In order to verify the cinder-volume service is actually running. This can be done by sourcing the "overcloudrc" file on the director,
and then run "openstack volume service list"
```
(overcloud) [stack@manager1 ~]$ openstack volume service list
+------------------+-------------------------------+------+---------+-------+----------------------------+
| Binary           | Host                          | Zone | Status  | State | Updated At                 |
+------------------+-------------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | overcloud-controller-1        | nova | enabled | up    | 2022-03-29T10:29:01.000000 |
| cinder-scheduler | overcloud-controller-2        | nova | enabled | up    | 2022-03-29T10:29:01.000000 |
| cinder-scheduler | overcloud-controller-0        | nova | enabled | up    | 2022-03-29T10:29:02.000000 |
| cinder-volume    | hostgroup@nimble              | nova | enabled | up    | 2022-03-25T08:54:43.000000 |
| cinder-backup    | overcloud-controller-2        | nova | enabled | up    | 2022-03-29T10:29:01.000000 |
+------------------+-------------------------------+------+---------+-------+----------------------------+

```
As it is clear in above output cinder-volume service is up and running.

3.2 In a production environment with three controller nodes, it will be necessary to determine which node is running the cinder-volume service. One
  way to do that is to run "pcs status" on any controller, and the output will reveal where the cinder-volume service is currently running.
```
[heat-admin@overcloud-controller-0 ~]$ sudo pcs status|grep openstack-cinder-volume-podman-0
    * openstack-cinder-volume-podman-0  (ocf::heartbeat:podman):        Started overcloud-controller-1
```
It is clear from the above output of "pcs status" that cinder-volume service is running on controller 1 where other controllers are controller 0 and controller 2.

3.3. Go to the controller node on which cinder-volume service is running and execute "sudo podman exec -it openstack-cinder-volume-podman-0 bash" and verify that the backend details are visible in ```/etc/cinder/cinder.conf``` in the cinder-volume container

Given below is an example of iSCSI backend details.
```
[nimble_iscsi]
image_volume_cache_enabled=True
nimble_pool_name=default
nimble_subnet_label=management
num_volume_device_scan_tries=10
san_ip=<3par_ip>
san_login=<3par_username>
san_password=<3par_password>
use_multipath_for_image_xfer=True
volume_backend_name=nimble_iscsi
volume_clear=zero
volume_driver=cinder.volume.drivers.nimble.NimbleISCSIDriver
backend_host=hostgroup
```
3.4 To verify that the test volume can be successfully created go the undercloud and source "overcloudrc" and execute below command to create "test" volume.
cinder create --name  --volume-type <volume-type> <size>
And "cinder list" to verify the results.    
```
    [stack@cld13b4 ~]$ source overcloudrc
(overcloud) [stack@manager1 ~]$ cinder create --name test --volume-type nimble_1 2
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | false                                |
| consistencygroup_id            | None                                 |
| created_at                     | 2022-03-29T10:48:16.000000           |
| description                    | None                                 |
| encrypted                      | False                                |
| id                             | 529d447f-fd78-4496-8760-8745c02d7ff8 |
| metadata                       | {}                                   |
| migration_status               | None                                 |
| multiattach                    | False                                |
| name                           | test                                 |
| os-vol-host-attr:host          | None                                 |
| os-vol-mig-status-attr:migstat | None                                 |
| os-vol-mig-status-attr:name_id | None                                 |
| os-vol-tenant-attr:tenant_id   | af1300d0e9044f21800042ce25aecfa2     |
| replication_status             | None                                 |
| size                           | 2                                    |
| snapshot_id                    | None                                 |
| source_volid                   | None                                 |
| status                         | creating                             |
| updated_at                     | None                                 |
| user_id                        | 44426fbd9b0441bb8c60ce1526bf6237     |
| volume_type                    | nimble                             |
+--------------------------------+--------------------------------------+
(overcloud) [stack@manager1 ~]$ cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+--------------------------------------+
| ID                                   | Status    | Name         | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+-----------+--------------+------+-------------+----------+--------------------------------------+
| 529d447f-fd78-4496-8760-8745c02d7ff8 | available | test         | 2    | nimble    | false    |                                      |
+--------------------------------------+-----------+--------------+------+-------------+----------+--------------------------------------+
(overcloud) [stack@cld13b4 ~]$

```