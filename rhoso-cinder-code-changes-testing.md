# RHOSO Cinder Code Changes Testing Guide

## Overview

This guide demonstrates how to test Cinder code changes in Red Hat OpenStack Services on OpenShift (RHOSO) using an NFS-mounted volume. This approach allows you to modify Cinder code without rebuilding container images.

**Reference:** https://openstack-k8s-operators.github.io/edpm-ansible/testing_with_ansibleee.html

## Architecture Flow

```
OpenStackControlPlane CR (extraMounts)
    ↓ uses
PersistentVolumeClaim (cinder-code-pvc)
    ↓ binds to
PersistentVolume (cinder-code-pv)
    ↓ mounts
NFS Export (/home/zuul/cinder-code)
    ↓ contains
Modified Cinder Code
```

## Test Environment

- **NFS Server Host:** sharkxx
- **NFS Export Path:** `/home/zuul/cinder-code`
- **CRC VM IP:** 192.168.130.11
- **NFS Server IP:** 192.168.130.1 (adjust to your sharkxx IP reachable from CRC)
- **Namespace:** openstack

## Prerequisites

- OpenShift CRC environment running
- OpenStack deployed via openstack-operator
- NFS utilities installed on the NFS server host
- Network connectivity between CRC VM and NFS server

## Step-by-Step Instructions

### Step 1: Copy Cinder Code from Pod

Extract the current Cinder code from a running cinder-volume pod to your directory:

```bash
mkdir -p ~/cinder-code
oc cp cinder-volume-ontap-iscsi-0:/usr/lib/python3.9/site-packages/cinder ~/cinder-code
```

**Note:** Adjust the pod name based on your deployment (e.g., `cinder-volume-lvm-iscsi-0`).

### Step 2: Configure NFS Server

#### 2.1 Create NFS Export

Export the directory so NFS clients in the specified subnet can access it:

```bash
sudo mkdir -p ~/cinder-code
echo "/home/zuul/cinder-code 192.168.130.0/24(rw,sync,no_root_squash,insecure)" | sudo tee /etc/exports
sudo exportfs -r
```

**Important Options Explained:**
- `rw` - Read-write access
- `sync` - Synchronous writes
- `no_root_squash` - Allow root access from clients (required for pod mounts)
- `insecure` - Allow connections from ports > 1024

#### 2.2 Start Required Services

Ensure firewalld and nfs-server services are running:

```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
```

#### 2.3 Configure Firewall (CRC + NFS on Same Hypervisor)

If your CRC VM and NFS server are on the same hypervisor, add a firewall rule to allow NFS traffic:

```bash
sudo nft add rule inet firewalld filter_IN_libvirt_pre accept
```

Alternatively, you can open specific NFS ports:

```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

### Step 3: Create StorageClass, PersistentVolume and PersistentVolumeClaim

#### 3.1 Create StorageClass

First, create a StorageClass for manual provisioning:

```bash
cat > ~/cinder-code-sc.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cinder-code-sc
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
EOF
```

Apply the StorageClass:

```bash
oc apply -f ~/cinder-code-sc.yaml
```

Verify the StorageClass was created:

```bash
oc get storageclass cinder-code-sc
```

#### 3.2 Create PersistentVolume and PersistentVolumeClaim

Now create a YAML file defining the PV and PVC:

```bash
cat > ~/openstack-code-storage.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cinder-code-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cinder-code-sc
  mountOptions:
    - vers=4
  nfs:
    server: 192.168.130.1   # <-- Update to your sharkxx IP reachable from CRC
    path: /home/zuul/cinder-code
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cinder-code-pvc
  namespace: openstack
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: cinder-code-sc
  resources:
    requests:
      storage: 5Gi
EOF
```

Apply the PV and PVC configuration:

```bash
oc apply -f ~/openstack-code-storage.yaml
```

Verify the PV and PVC are bound:

```bash
oc get pv cinder-code-pv
oc get pvc -n openstack cinder-code-pvc
```

Expected output shows `STATUS: Bound`.

### Step 4: Add extraMounts to OpenStackControlPlane CR

Edit your OpenStackControlPlane CR to add the `extraMounts` section at the `cinder.template` level (same level as `cinderVolumes`, `cinderScheduler`, and `cinderAPI`):

```bash
oc edit openstackcontrolplane -n openstack
```

Add the following configuration:

```yaml
spec:
  cinder:
    template:
      cinderAPI:
        replicas: 1
        resources: {}
      cinderScheduler:
        replicas: 1
        resources: {}
      cinderVolumes:
        lvm-iscsi:
          customServiceConfig: |
            [lvm]
            image_volume_cache_enabled=false
            volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
            ...
          nodeSelector:
            openstack.org/cinder-lvm: ""
          replicas: 1
          resources: {}
      extraMounts:
      - name: cinder-code
        region: r1
        extraVol:
          - propagation:
            - CinderVolume
            extraVolType: Cinder
            volumes:
            - name: cinder-code
              persistentVolumeClaim:
                claimName: cinder-code-pvc
            mounts:
            - name: cinder-code
              mountPath: /usr/lib/python3.9/site-packages/cinder
              readOnly: true
      databaseAccount: cinder
      databaseInstance: openstack
```

**Important:**
- `extraMounts` must be at the same indentation level as `cinderVolumes`, not inside it
- To verify where `extraMounts` is supported in the CRD schema:
  ```bash
  oc get crd openstackcontrolplanes.core.openstack.org -o yaml | grep -A 50 extraMounts
  ```

Save and exit. The operator will automatically restart the cinder-volume pods with the new mount.

### Step 5: Verify the Configuration

#### 5.1 Check Pod Restart

Monitor the cinder-volume pods to ensure they restart:

```bash
oc get pods -n openstack -l component=cinder-volume -w
```

#### 5.2 Verify Mount Inside Pod

Log into the cinder-volume pod and verify the code is mounted:

```bash
oc rsh -n openstack cinder-volume-lvm-iscsi-0
cd /usr/lib/python3.9/site-packages/cinder
ls -la
exit
```

You should see your modified Cinder code.

## Troubleshooting

### Issue: Code Not Appearing in Pod

If the modified code is not visible inside the pod, check the following:

#### 1. Verify Pod Restarted

```bash
oc get pods -n openstack -l component=cinder-volume
```

Check the `AGE` column. If pods haven't restarted recently, there's a configuration issue.

#### 2. Check OpenStackControlPlane CR

Verify `extraMounts` is present in the actual spec (not just in annotations):

```bash
oc get openstackcontrolplane -n openstack -o yaml | grep -A 20 "extraMounts"
```

If it's only in `kubectl.kubernetes.io/last-applied-configuration` annotation but not in the spec, the configuration is being rejected.

#### 3. Check StatefulSet Volume Configuration

Verify the mount is present in the StatefulSet:

```bash
oc get statefulset -n openstack -l component=cinder-volume -o yaml | grep -A 10 "cinder-code"
```

Look for:
```yaml
- mountPath: /usr/lib/python3.9/site-packages/cinder
  name: cinder-code
  readOnly: true
```

#### 4. Inspect Pod Volume Mounts

Check the pod description for volume mounts:

```bash
oc describe pod -n openstack cinder-volume-lvm-iscsi-0
```

Under `Mounts:` section, look for:
```
/usr/lib/python3.9/site-packages/cinder from cinder-code (ro)
```

#### 5. Check NFS Connectivity

Test NFS mount from a test pod:

```bash
oc run -n openstack nfs-test --rm -it --image=busybox -- sh
# Inside the pod:
mount -t nfs -o vers=4 192.168.130.1:/home/zuul/cinder-code /mnt
ls /mnt
exit
```

#### 6. Check Operator Logs

Review operator logs for errors:

```bash
oc logs -n openstack-operators deployment/openstack-operator-controller-manager --tail=100 | grep -i cinder
```

### Issue: PVC Not Binding

If the PVC remains in `Pending` state:

```bash
oc describe pvc -n openstack cinder-code-pvc
```

Common causes:
- NFS server not accessible from CRC
- Incorrect IP address in PV spec
- NFS service not running
- Firewall blocking NFS ports

Test NFS connectivity from your workstation:

```bash
showmount -e 192.168.130.1
```

## Making Code Changes

After the initial setup, to test code changes:

1. **Modify code** on the NFS server:
   ```bash
   vi ~/cinder-code/volume/drivers/lvm.py
   ```

2. **Restart the cinder-volume pod** to reload the code:
   ```bash
   oc delete pod -n openstack cinder-volume-lvm-iscsi-0
   ```

3. **Verify changes** took effect by checking logs or testing functionality

**Note:** Since the mount is `readOnly: true` in the pod, all modifications must be made on the NFS server host.

## Cleanup

To remove the test configuration:

```bash
# Remove extraMounts from OpenStackControlPlane CR
oc edit openstackcontrolplane -n openstack
# (Remove the extraMounts section)

# Delete PVC and PV
oc delete -f ~/openstack-code-storage.yaml

# Delete StorageClass
oc delete -f ~/cinder-code-sc.yaml

# Stop NFS export (on NFS server)
sudo sed -i '/\/home\/zuul\/cinder-code/d' /etc/exports
sudo exportfs -r
```

## Additional Notes

- The `readOnly: true` setting prevents accidental code modification from within the pod
- Changes to code on the NFS server are immediately visible after pod restart
- This method works for any OpenStack service that supports `extraMounts`
- For production deployments, always build proper container images instead of using this approach
- The `propagation: [CinderVolume]` ensures the mount is only applied to cinder-volume pods, not cinder-api or cinder-scheduler

## References

- [OpenStack K8s Operators - Testing with AnsibleEE](https://openstack-k8s-operators.github.io/edpm-ansible/testing_with_ansibleee.html)
- [Red Hat OpenStack Services on OpenShift Documentation](https://access.redhat.com/documentation/en-us/red_hat_openstack_services_on_openshift)
