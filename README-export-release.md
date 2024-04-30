
## Export Final Release(experimental)
referece to https://bosh.io/docs/compiled-releases/

TODO: the exported release has dependency to os, version 


### steps

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
