

export VERSION=1.0.0

bosh create-release --final --version=$VERSION

bosh create-release releases/custom-vm-release/custom-vm-release-$VERSION.yml --tarball ./custom-vm-release-$VERSION.tgz

bosh upload-release ./custom-vm-release-$VERSION.tgz

bosh update-runtime-config --name=custom-vm-release-addon ./runtimeconfig.yml

bosh config --type=runtime --name=custom-vm-release-addon

bosh -d TARGET_DEPLOYMENT manifest > manifest.yml

bosh -d TARGET_DEPLOYMENT deploy manifest.yml