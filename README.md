WARNING
===============
this repo is not supported by VMware

Custom VM bosh release for customizing routes
===============

if you customized the VM network (especially under /etc/systemd/network or route) via runtime config(using os-conf release), it will be persisted in case of:
- vm is provisioned
- vm is rebooted

but it will be lost in case of follwing:
- if director vm is recreated, then bosh-agent will reset the all custom network config.
- if the custom network config files under /etc/systemd/network are deleted by any reason, it will not be restored until the os-conf job is executed via rebooting or recreating the VM.

this document explains how to customize network route config and preserve them using runtime config and custom bosh release.

eventually, all required custom config will be in bosh runtime-config file: 
```
addons:
- name: custom-vm-release-addon
  include:
    deployments:
    - p-isolation-segment-is1-6e18f0c63108927910d4
  jobs:
  - name: custom-vm-routing
    release: custom-vm-release
    properties:
      default_gw: 192.168.0.1
      routes:
      - device: eth1
        cidr: 192.4.1.0/24
        gateway: 192.168.40.1
      - device: eth1
        cidr: 192.5.1.0/24
        gateway: 192.168.40.1
```        
and it will be configured to the target VM on additional config under subdirectory named after each NIC name.
```
/etc/systemd/network/10_eth1.network.d/custom-network.conf

[Route]
Gateway=192.168.40.1
Destination=192.4.1.0/24

[Route]
Gateway=192.168.40.1
Destination=192.5.1.0/24

```

note that, this repo has been tested in TAS 4.x on ubuntu jammy stemcell.

Usage
-----

## Steps summary

```
bosh create-release --final --version=1.0.0

bosh create-release releases/custom-vm-release/custom-vm-release-1.0.0.yml \
        --tarball ./custom-vm-release-1.0.0.tgz

bosh upload-release ./custom-vm-release-1.0.0.tgz

bosh update-runtime-config --name=custom-vm-release-addon ./runtimeconfig.yml

bosh config --type=runtime --name=custom-vm-release-addon

bosh -d TARGET_DEPLOYMENT manifest > manifest.yml

bosh -d TARGET_DEPLOYMENT deploy manifest.yml
```


## Detailed steps

### (optional) Build Final Release
you can use the existing release, but you can build your own release if required. unzip this repo to any environment where bosh command is available such as ops-manager.
```
cd custom-vm-release
```
following command will build a new release under current directory.
```
bosh create-release --final --version=1.0.0
```

it will generates files under following directory: 
- dev_releases/
- releases/
- .final_builds/

you can pick a next version by listing the existing version from ./releases/custom-vm-release/.
and if there is a version conflict, you can clean up by deleting specific build version or remove the related files above.
 or delete the above folders to clean them all. 

now package a new release with specific version info locally.
```
bosh create-release releases/custom-vm-release/custom-vm-release-1.0.0.yml \
        --tarball ./custom-vm-release-1.0.0.tgz
```
### Upload release
login in to bosh env and upload the release
```
bosh env
bosh upload-release ./custom-vm-release-1.0.0.tgz
```
verify uploaded release
```
bosh releases | grep custom
custom-vm-release              	1.0.0        	23e13f4

```

note that if you run again without version, it will increase version to next patch locally. and you can clean up by deleting specific build version related files from above folder.
```
bosh create-release --final 
```

### Create runtime config
add custom route config info into runtime config, [runtimeconfig.yml](runtimeconfig.yml)
```
releases:
- name: custom-vm-release
  version: 1.0.0

addons:
- name: custom-vm-release-addon
  include:
    deployments:
    - p-isolation-segment-is1-6e18f0c63108927910d4
  jobs:
  - name: custom-vm-routing
    release: custom-vm-release
    properties:
      default_gw: 192.168.0.1
      routes:
      - device: eth1
        cidr: 192.4.1.0/24
        gateway: 192.168.40.1
      - device: eth1
        cidr: 192.5.1.0/24
        gateway: 192.168.40.1
      - device: eth1
        cidr: 192.3.1.0/24
        gateway: 192.168.40.1
```

update runtime config and verify. optionally delete old config if need
```
bosh update-runtime-config --name=custom-vm-release-addon ./runtimeconfig.yml
bosh config --type=runtime --name=custom-vm-release-addon
bosh delete-config --type=runtime --name=custom-vm-release-addon
```


### Apply runtime config to deployment
use any existing deployment
```
bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 manifest > manifest.yml
bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 deploy manifest.yml
```

### Verify artifacts created on VM
then, following artifacts are created in the deployed VM

```
monit summary
...
Process 'system-metrics-agent'      running
Process 'custom-vm-routing'         running
```

```
/var/vcap/jobs/custom-vm-routing/bin
configure_routes
monitor_routes
routing_ctl
```

### Verify the applied custom network config

#### Default network config by bosh-agent.
```
/etc/systemd/network/10_eth0.network
/etc/systemd/network/10_eth1.network
```

/etc/systemd/network/10_eth0.network
```
# Generated by bosh-agent
[Match]
Name=eth0

[Address]
Address=192.168.0.76/24
Broadcast=192.168.0.255

[Network]
Gateway=192.168.0.1
DNS=192.168.0.5
```
/etc/systemd/network/10_eth1.network
```
# Generated by bosh-agent
[Match]
Name=eth1

[Address]
Address=192.168.40.11/24

[Network]
DNS=192.168.0.5
```

#### Verify custom network config created by this custom bosh release

```
find /etc/systemd/network

/etc/systemd/network/10_eth0.network
/etc/systemd/network/10_eth1.network
/etc/systemd/network/10_eth1.network.d
/etc/systemd/network/10_eth1.network.d/custom-network.conf
```

/etc/systemd/network/10_eth1.network.d/custom-network.conf
```
[Route]
Gateway=192.168.40.1
Destination=192.4.1.0/24

[Route]
Gateway=192.168.40.1
Destination=192.5.1.0/24

[Route]
Gateway=192.168.40.1
Destination=192.3.1.0/24
```


and route config.
```
ip route show

default via 192.168.0.1 dev eth0 proto static
10.255.0.0/16 dev silk-vtep proto kernel scope link src 10.255.206.0
10.255.119.0/24 via 10.255.119.0 dev silk-vtep src 10.255.206.0
10.255.136.0/24 via 10.255.136.0 dev silk-vtep src 10.255.206.0
10.255.233.0/24 via 10.255.233.0 dev silk-vtep src 10.255.206.0
192.3.1.0/24 via 192.168.40.1 dev eth1 proto static
192.4.1.0/24 via 192.168.40.1 dev eth1 proto static
192.5.1.0/24 via 192.168.40.1 dev eth1 proto static
192.168.0.0/24 dev eth0 proto kernel scope link src 192.168.0.76
192.168.40.0/24 dev eth1 proto kernel scope link src 192.168.40.11
```

#### Check release logs/Troubleshooting
for any reason, the custom network config file is missing, /etc/systemd/network/10_ethX_network.d/custom-network.conf, then monit `custom-vm-routing` monit kicks in and re-configure the setting by running `/var/vcap/jobs/custom-vm-routing/bin/configure_routes`
the activity is found in the log file.
```
isolated_diego_cell_is1/a4ccb2fa-af7a-48d1-b9de-1e8bb2a70be6:/var/vcap/bosh/log# tail -f /var/vcap/sys/log/custom-vm-routing/custom-vm-routing.log
[monitor_routes]  start monitoring network routes
[monitor_routes] WARNING: missing route config file: /etc/systemd/network/10_eth1.network.d/custom-network.conf
[configure_routes] reconfiguring network routes...
[configure_routes] creating network route config /etc/systemd/network/10_eth1.network.d/custom-network.conf
[configure_routes] creating network route config /etc/systemd/network/10_eth1.network.d/custom-network.conf
[configure_routes] creating network route config /etc/systemd/network/10_eth1.network.d/custom-network.conf
[configure_routes]  restarting network
```

### Reference
- https://bosh.io/docs/create-release/
- https://github.com/cloudfoundry/os-conf-release
- https://github.com/rakutentech/bosh-routing-release
