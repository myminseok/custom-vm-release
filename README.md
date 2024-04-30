
Custom VM bosh release for customizing routes
===============
this document explains how to customize network route config using runtime config and custom bosh release.

this release supports keeping the custom routes in case of:
- vm is provisioned
- vm is rebooted
- if director vm is recreated and all customer network on bosh deployed vm will be reset by bosh-agent. this release can persist the custom network routes
- deleting the custom network config file /etc/systemd/network/10_ethx_network.d/custom-network.conf

all required custom config will be set in bosh runtime-config file. 

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
```        
and it will be configured to the target VM on additinal config under subdirectory named aftereach NIC name.
```
/etc/systemd/network/10_eth1.network.d/custom-network.conf

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


Usage
-----

### build


#### build steps summary

```
cd custom-vm-release

bosh create-release --force

bosh upload-release

bosh releases | grep routing

bosh update-runtime-config --name=custom-vm-release-is ./custom-vm-release-runtimeconfig.yml

bosh config --type=runtime --name=custom-vm-release-is

bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 manifest > iso-manifest.yml

bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 deploy iso-manifest.yml
```

#### Detailed build steps
unzip this repo to any environment where bosh command is available such as opsmanager. build
```
cd custom-vm-release
```
then build dev release under current directory

```
bosh create-release --force

Adding job 'custom-vm-routing/d76f674f807f766d1bf37004e400491925ce9b7b1fa653788b21f17087e8b214'...
Added job 'custom-vm-routing/d76f674f807f766d1bf37004e400491925ce9b7b1fa653788b21f17087e8b214'
Added dev release 'custom-vm-release/0+dev.11'

Name         custom-vm-release
Version      0+dev.11
Commit Hash  96502c9+

Job                                                                                 Digest                                                                   Packages
custom-vm-routing/d76f674f807f766d1bf37004e400491925ce9b7b1fa653788b21f17087e8b214  sha256:94d22244bda0ce957fa238069cd0624ab7816b3a0fdc2e146990a6ce137b822a  -

1 jobs

Package  Digest  Dependencies

0 packages

Succeeded
```

now login in to bosh env and upload the release
```
bosh upload-release
Using environment '192.168.0.55' as client 'ops_manager'

[-----------------------------------------------------------------] 100.00%   0s
Task 1655

Task 1655 | 08:36:13 | Extracting release: Extracting release (00:00:00)
Task 1655 | 08:36:13 | Verifying manifest: Verifying manifest (00:00:00)
Task 1655 | 08:36:13 | Resolving package dependencies: Resolving package dependencies (00:00:00)
Task 1655 | 08:36:13 | Creating new jobs: custom-vm-routing/d76f674f807f766d1bf37004e400491925ce9b7b1fa653788b21f17087e8b214 (00:00:00)
Task 1655 | 08:36:13 | Release has been created: custom-vm-release/0+dev.11 (00:00:00)

Task 1655 Started  Tue Apr 30 08:36:13 UTC 2024
Task 1655 Finished Tue Apr 30 08:36:13 UTC 2024
Task 1655 Duration 00:00:00
Task 1655 done

Succeeded
```

```
bosh releases | grep custom
custom-vm-release              	0+dev.11               	96502c9+
```


### create runtime config
add custom route config info into runtime config, custom-vm-release-runtimeconfig.yml
```
releases:
- name: custom-vm-release
  version: 0+dev.11

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
bosh update-runtime-config --name=custom-vm-release-is ./custom-vm-release-runtimeconfig.yml
bosh config --type=runtime --name=custom-vm-release-is
bosh delete-config --type=runtime --name=custom-vm-release-is
```


### Apply runtime config to deployment
```
bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 manifest > iso-manifest.yml
bosh -d p-isolation-segment-is1-6e18f0c63108927910d4 deploy iso-manifest.yml
```

### bosh release artifacts
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

### custom network config

####  default network config by bosh-agent.
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

####  Custom network config by this custom bosh release

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


### Reference
- https://bosh.io/docs/create-release/
- https://github.com/cloudfoundry/os-conf-release
- https://github.com/rakutentech/bosh-routing-release