# Nimble Deployment Guide for RHOSP16.2

## Overview

This page provides detailed steps on how to enable the containerization of Nimble Cinder driver on top of the OSP Cinder images.
It also contains steps to deploy & configure Nimble backends for RHOSP16.2.

## Prerequisites

* Red Hat OpenStack Platform 16.2 with RHEL 8.4.

* Nimble array 6.0.0 or higher.

## Steps

### 1.	Prepare the Environment Files for containers and cinder backend

#### 1.1 Environment File for cinder-volume container

To use Nimble hardware as a Block Storage back end, cinder-volume container needs to be deployed.

Procedure

Create a new container images file for your overcloud:

```
openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameter-nimble.yaml
```

Edit the containers-prepare-parameter-nimble.yaml file and include the customized containers depending upon your requirements.

```
parameter_defaults:
  ContainerImagePrepare:
  - push_destination: true
    set:
      namespace: registry.redhat.io/rhosp-rhel8
      name_prefix: openstack-
      name_suffix: ''
      tag: '16.2'
    tag_from_label: '{version}-{release}'        ...
```

Add the authentication details for the registry.connect.redhat.com registry to the ContainerImageRegistryCredentials parameter:

```
parameter_defaults:
  ContainerImageRegistryCredentials:
    registry.redhat.io:
      [service account username]: [service account password]
    registry.connect.redhat.com:
      [service account username]: [service account password]
```

Save the containers-prepare-parameter-nimble.yaml file.

Include the containers-prepare-parameter-nimble.yaml file with any deployment commands, such as as openstack overcloud deploy:

```
openstack overcloud deploy --templates
    ...
    -e containers-prepare-parameter-nimble.yaml
    ...
```

IMPORTANT:

The containers-prepare-parameter-nimble.yaml file replaces the standard containers-prepare-parameter.yaml file in your overcloud deployment. Do not include the standard containers-prepare-parameter.yaml file in your overcloud deployment. Retain the standard containers-prepare-parameter.yaml file for your undercloud installation and updates.



#### 1.2 Environment File for cinder backend

The environment file is an OSP director environment file. The environment file contains the settings for each backend you want to define.

Create the environment file “cinder-nimble-[iscsi|fc].yaml” under /home/stack/templates/ with below parameters and other backend details.

```
parameter_defaults:
  CinderEnableIscsiBackend: false
  ControllerExtraConfig:
```

Sample files for iSCSI backend is available in [templates](https://github.com/mohdadilntl/hpe-nimble-cinder-rhosp16.2/blob/master/templates) folder for reference.

#### Additional Help

For further details of Nimble cinder driver, kindly refer documentation [here](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/nimble-volume-driver.html)


### 2.	Deploy the overcloud and configured backends

After creating ```cinder-nimble-[iscsi|fc].yaml``` file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option.
Use the ```-e``` option to include the environment file ```cinder-nimble-[iscsi|fc].yaml```.

The order of the environment files (.yaml) is important as the parameters and resources defined in subsequent environment files take precedence.

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates \
    -e /home/stack/templates/node-info.yaml \
    -e /home/stack/containers-prepare-parameter-nimble.yaml \
    -e /home/stack/templates/cinder-nimble-[iscsi|fc].yaml \
    --ntp-server <ntp_server_ip> \
    --debug
```

### 3.	Verify the configured changes

3.1	SSH to controller node from undercloud and check the docker process for cinder-volume
```
(overcloud) [heat-admin@overcloud-controller-0 ~]$ sudo podman ps | grep cinder
7615e1056547  manager1.ctlplane.gse.com:8787/hpe3parcinder/openstack-cinder-volume-hpe3parcinder16-2:latest  /bin/bash /usr/lo...  12 days ago  Up 4 days ago           openstack-cinder-volume-podman-0
c2c61e847664  manager1.ctlplane.gse.com:8787/rhosp-rhel8/openstack-cinder-backup:16.2                        /bin/bash /usr/lo...  12 days ago  Up 4 days ago           openstack-cinder-backup-podman-0
650d205dfa69  manager1.ctlplane.gse.com:8787/rhosp-rhel8/openstack-cinder-scheduler:16.2                     kolla_start           12 days ago  Up 4 days ago           cinder_scheduler
47b0758c5c8a  manager1.ctlplane.gse.com:8787/rhosp-rhel8/openstack-cinder-api:16.2                           kolla_start           12 days ago  Up 4 days ago           cinder_api_cron
a304f4cee9d8  manager1.ctlplane.gse.com:8787/rhosp-rhel8/openstack-cinder-api:16.2                           kolla_start           12 days ago  Up 4 days ago           cinder_api
```

3.2.	Verify that the backend details are visible in ```/etc/cinder/cinder.conf``` in the cinder-volume container

Given below is an example of iSCSI backend details. Similar entries should be observed for FC backend too.

```
[nimble_iscsi]
Image_volume_cache_enabled=True
Nimble_pool_name=default
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
