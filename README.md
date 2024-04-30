
Custom VM bosh release for customizing routes
===============
this document explains how to customize network route config using runtime config and custom bosh release.

this release supports keeping the custom routes in case of:
- vm is provisioned
- vm is rebooted
- if director vm is recreated and all customer network on bosh deployed vm will be reset by bosh-agent. this release can persist the custom network routes
- deleting the custom network config file /etc/systemd/network/10_ethX_network.d/custom-network.conf

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
and it will be configured to the target VM on additiㅐnal config under subdirectory named after each NIC name.
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

note that, this repo has been tested in TAS 4.x on ubuntu jammy stemcell.

Usage
-----

### Build release


#### Build steps summary

```
cd custom-vm-release

bosh create-release --force

bosh upload-release

bosh releases | grep custom

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


### Create runtime config
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

### Artifacts to be created on VM
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

### Custom network config

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

#### Custom network config by this custom bosh release

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

### recovering custom config.
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


### Export release(experimental)

bosh director should be ready.

```
bosh env
```

build final relase
```
bosh create-release --final --version=1.0.0
```
to clean release delete version from:
- dev_releases/
- releases/
- .final_builds/
- /tmp/custom-vm-release/(optional)

deploy complilation deployment. it does not allocate any resources when deployed.  reference https://bosh.io/docs/compiled-releases/
```
bosh -d compilation-deployment deploy complication-deployment.yml
```

verify os version info
```
bosh stemcells

Using environment '192.168.0.55' as client 'ops_manager'
Name                                     Version  OS            CPI                   CID
bosh-vsphere-esxi-ubuntu-jammy-go_agent  1.360*   ubuntu-jammy  ef3542b599f10b338ad6  sc-416cde69-ea80-49ab-a9c3-1353401abff0

```

export release with the correct os/version from above
```
bosh -d compilation-deployment export-release custom-vm-release/1.0.0 ubuntu-jammy/1.360

Using environment '192.168.0.55' as client 'ops_manager'

Using deployment 'compilation-deployment'

Task 1732

Task 1732 | 13:07:52 | Preparing package compilation: Finding packages to compile (00:00:00)
Task 1732 | 13:07:52 | copying jobs: custom-vm-routing/d76f674f807f766d1bf37004e400491925ce9b7b1fa653788b21f17087e8b214 (00:00:00)

Task 1732 Started  Tue Apr 30 13:07:52 UTC 2024
Task 1732 Finished Tue Apr 30 13:07:52 UTC 2024
Task 1732 Duration 00:00:00
Task 1732 done

Downloading resource '71ba8056-657a-4946-98d7-7afc77ad7ef9' to '/home/ubuntu/workspace/custom-vm-release/custom-vm-release-1.0.0-ubuntu-jammy-1.360-20240430-130752-233248917.tgz'...

```



### Reference
- https://bosh.io/docs/create-release/
- https://github.com/cloudfoundry/os-conf-release
- https://github.com/rakutentech/bosh-routing-release
