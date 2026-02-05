# RHOSP Cinder Code Changes Testing Guide

This guide demonstrates methods to test Cinder code changes in Red Hat OpenStack Platform (RHOSP) using TripleO/Director deployments with Pacemaker-managed containers.

## Method 1: Using podman mount and update the container code

**Note:** Requires container restart (not recreation)

### Steps

#### 1. Unmanage the container from Pacemaker

Prevent Pacemaker from recreating the container during testing:

```bash
sudo pcs resource unmanage openstack-cinder-volume
```

#### 2. Mount the container filesystem

```bash
CINDER_CONTAINER=$(sudo podman ps -q --filter name=cinder-volume)
MOUNT=$(sudo podman mount $CINDER_CONTAINER)
```

#### 3. Make code changes directly on the host

```bash
sudo vi $MOUNT/usr/lib/python3.9/site-packages/cinder/volume/manager.py
```

Edit your code as needed.

#### 4. Unmount the container

```bash
sudo podman unmount $CINDER_CONTAINER
```

#### 5. Restart the container to pick up changes

```bash
sudo podman restart $CINDER_CONTAINER
```

#### 6. Test your changes

Run your tests...

#### 7. Re-enable Pacemaker management

After testing, make the container managed as a Pacemaker resource again:

```bash
sudo pcs resource manage openstack-cinder-volume
```

---

## Method 2: New cinder-volume image with code changes

**Note:** Requires container recreation

This method creates a new container image with your code changes and uses it for the cinder-volume container.

### Steps

#### 1. Create backup tags for the current image

```bash
CINDER_CONTAINER=$(sudo podman ps -q --filter name=cinder-volume)
CURRENT_IMAGE=$(sudo podman inspect $CINDER_CONTAINER --format '{{.ImageName}}')
sudo podman tag $CURRENT_IMAGE cluster.common.tag/cinder-volume:pcmkoriginal
```

**Optional:** Create a timestamped backup:

```bash
sudo podman tag $CURRENT_IMAGE cluster.common.tag/cinder-volume:pcmk-backup-$(date +%Y%m%d)
```

#### 2. Verify the backup tag is created

```bash
sudo podman images | grep cluster.common.tag
```

**Example output:**

```
cluster.common.tag/cinder-volume    pcmkoriginal     9d7e73f3c621  3 weeks ago  1.09 GB
cluster.common.tag/cinder-volume    pcmklatest       9d7e73f3c621  3 weeks ago  1.09 GB
```

#### 3. Modify the running container

```bash
sudo podman exec -it $CINDER_CONTAINER /bin/bash
vi /usr/lib/python3.9/site-packages/cinder/volume/manager.py
# Make your code changes
exit
```

#### 4. Commit the changes to a local image

```bash
sudo podman commit $CINDER_CONTAINER localhost/cinder_modified:latest
```

#### 5. Verify the modified local image is created

```bash
sudo podman images | grep cinder
```

**Example output:**

```
localhost/cinder_modified                                                  latest           62b47e7dab22  3 minutes ago  1.09 GB
cluster.common.tag/cinder-volume                                           pcmklatest       9d7e73f3c621  3 weeks ago    1.09 GB
cluster.common.tag/cinder-volume                                           pcmkoriginal     9d7e73f3c621  3 weeks ago    1.09 GB
undercloud-0.ctlplane.redhat.local:8787/rhosp-rhel9/openstack-cinder-volume  17.1_20260112.1  9d7e73f3c621  3 weeks ago    1.09 GB
```

#### 6. (Optional) Compare original vs modified image

```bash
ORIGINAL_ID=$(sudo podman images --format "{{.ID}}" cluster.common.tag/cinder-volume:pcmkoriginal | head -1)
MODIFIED_ID=$(sudo podman images --format "{{.ID}}" localhost/cinder_modified:latest | head -1)

echo "Original: $ORIGINAL_ID"
echo "Modified: $MODIFIED_ID"

# See differences
sudo podman diff $MODIFIED_ID
```

#### 7. Tag the modified image with the pcmk "latest" name

```bash
sudo podman tag localhost/cinder_modified:latest cluster.common.tag/cinder-volume:pcmklatest
```

#### 8. Verify cluster.common.tag/cinder-volume is pointing to the new image

```bash
sudo podman images | grep cinder
```

**Example output:**

```
cluster.common.tag/cinder-volume                                           pcmklatest       62b47e7dab22  7 minutes ago  1.09 GB
localhost/cinder_modified                                                  latest           62b47e7dab22  7 minutes ago  1.09 GB
cluster.common.tag/cinder-volume                                           pcmkoriginal     9d7e73f3c621  3 weeks ago    1.09 GB
undercloud-0.ctlplane.redhat.local:8787/rhosp-rhel9/openstack-cinder-volume  17.1_20260112.1  9d7e73f3c621  3 weeks ago    1.09 GB
```

#### 9. Restart the cinder-volume service

```bash
sudo pcs resource restart openstack-cinder-volume
```

**Note:** This will recreate the cinder-volume container with a new container ID (using the modified image).

#### 10. Verify the running container is using the new image

```bash
CINDER_CONTAINER=$(sudo podman ps -q --filter name=cinder-volume)
sudo podman inspect $CINDER_CONTAINER | grep -A 1 '"Image":'
```

#### 11. Test your changes

Run your tests...

---

## Rollback Procedures (for Method 2)

### Option 1: Retag back to the original image

```bash
# Retag cluster.common.tag/cinder-volume:pcmklatest back to the original image
sudo podman tag cluster.common.tag/cinder-volume:pcmkoriginal cluster.common.tag/cinder-volume:pcmklatest
```

**Or using the image ID:**

```bash
sudo podman tag <image_id_of_pcmkoriginal> cluster.common.tag/cinder-volume:pcmklatest
```

**Example:**

```bash
sudo podman tag 9d7e73f3c621 cluster.common.tag/cinder-volume:pcmklatest
```

**Restart the service to use the original image:**

```bash
sudo pcs resource restart openstack-cinder-volume
```

**Note:** This will recreate the cinder-volume container with a new container ID (using the original image).

### Option 2: Instant rollback using pcs bundle update

If something goes wrong, use this alternative method:

```bash
# Disable the resource
sudo pcs resource disable openstack-cinder-volume

# Update the bundle to use the original image
sudo pcs resource bundle update openstack-cinder-volume container image=cluster.common.tag/cinder-volume:pcmkoriginal

# Re-enable the resource
sudo pcs resource enable openstack-cinder-volume
```

**Note:** After rollback using this method, the container will be recreated with a new container ID, but it will use the original cinder code. You can verify this:

```bash
sudo podman ps | grep cinder-vol
```

**Example output:**

```
bfa58e9e95c6  cluster.common.tag/cinder-volume:pcmkoriginal  /bin/bash /usr/lo...  13 minutes ago  Up 13 minutes  openstack-cinder-volume-podman-0
```

The container is now using the `pcmkoriginal` image with the original code.

---

## Method 3: Using containers-prepare-parameters.yaml to modify the cinder-volume image

**Note:** This method modifies the image during deployment/update using TripleO's container image preparation workflow.

### Overview

This approach uses `containers-prepare-parameters.yaml` to build a custom cinder-volume image with your code changes during the overcloud deployment or update process.

### Steps

#### 1. Create a working directory for cinder-volume modifications

```bash
mkdir /home/stack/cinder-volume-rbd
cd /home/stack/cinder-volume-rbd
```

#### 2. Create the Dockerfile

```bash
cat <<EOF > Dockerfile
FROM registry.redhat.io/rhosp-rhel9/openstack-cinder-volume:17.1
USER root
COPY rbd.patch /tmp
RUN yum install -y patch && patch -d /usr/lib/python3.9/site-packages -p1 </tmp/rbd.patch
USER cinder
EOF
```

**Alternative:** If patching fails, use direct file copy instead:

```bash
cat <<EOF > Dockerfile
FROM registry.redhat.io/rhosp-rhel9/openstack-cinder-volume:17.1
USER root
COPY rbd.py /usr/lib/python3.9/site-packages/cinder/volume/drivers/
USER cinder
EOF
```

#### 3. Copy your patch or modified Python file

Copy either your patch file:

```bash
cp /path/to/your/rbd.patch /home/stack/cinder-volume-rbd/
```

Or your modified Python file:

```bash
cp /path/to/your/modified/rbd.py /home/stack/cinder-volume-rbd/
```

#### 4. Update containers-prepare-parameters.yaml

Add the following configuration to your `containers-prepare-parameters.yaml`:

```yaml
  excludes:
    - cinder-volume
- set:
    name_prefix: openstack-
    name_suffix: ''
    namespace: registry.redhat.io/rhosp-rhel9
    rhel_containers: false
    tag: '17.1'
  includes:
    - cinder-volume
  modify_role: tripleo-modify-image
  modify_append_tag: "-rbd"
  modify_vars:
    tasks_from: modify_image.yml
    modify_dir_path: /home/stack/cinder-volume-rbd
  push_destination: true
```

**Important Notes:**

- **If credentials issue:** Replace `name_prefix`, `name_suffix`, `namespace`, and `tag` values with those from your existing `containers-prepare-parameters.yaml`
- **Reference:** See example at https://paste.opendev.org/show/bgCUCTs6aIsB86vz2iuX/

**Example of matching existing configuration:**

If your existing config has:

```yaml
  set:
    ceph_alertmanager_image: ose-prometheus-alertmanager
    ceph_alertmanager_namespace: registry.redhat.io/openshift4
    ceph_alertmanager_tag: v4.10
```

Then use those same registry details for consistency.

#### 5. Re-run the deployment command (update)

Run your overcloud deployment script which uses the updated `containers-prepare-parameters.yaml`:

```bash
./overcloud_deploy.sh
```

**Important:**
- Do **NOT** delete the current overcloud
- This is an update operation, not a fresh deployment
- The script will build the custom image and deploy it

#### 6. Monitor the image preparation process

Check the tripleo container image preparation logs:

```bash
tail -f /var/log/tripleo-container-image-prepare.log
```

If something fails, review this log for errors.

#### 7. Verify the custom image was created

After deployment completes, verify the custom image is being used:

```bash
# On a controller node
sudo podman images | grep cinder-volume

# Should see an image tagged with "-rbd"
# Example: undercloud-0.ctlplane:8787/rhosp-rhel9/openstack-cinder-volume:17.1-rbd
```

#### 8. Verify the running container is using the custom image

```bash
CINDER_CONTAINER=$(sudo podman ps -q --filter name=cinder-volume)
sudo podman inspect $CINDER_CONTAINER | grep -A 1 '"Image":'
```

#### 9. Test your changes

Run your tests...

### Rollback for Method 3

To rollback, remove the custom configuration from `containers-prepare-parameters.yaml`:

1. **Edit containers-prepare-parameters.yaml:**

   Remove or comment out the custom cinder-volume configuration:

   ```yaml
   # Comment out or remove:
   # excludes:
   #   - cinder-volume
   # - set:
   #     ...
   ```

2. **Re-run the deployment:**

   ```bash
   ./overcloud_deploy.sh
   ```

   This will revert to the standard cinder-volume image.


---

## Comparison: Method 1 vs Method 2 vs Method 3

| Feature | Method 1 (podman mount) | Method 2 (new image) | Method 3 (containers-prepare) |
|---------|------------------------|----------------------|-------------------------------|
| **Container recreation** | No (restart only) | Yes (new container ID) | Yes (deployment update) |
| **Persistence** | Lost on container recreation | Persisted in new image | Persisted in registry |
| **Rollback complexity** | Re-apply original files | Simple image retag | Deployment update |
| **Pacemaker management** | Must unmanage/manage | Managed by Pacemaker | Managed by Pacemaker |
| **Deployment integration** | None | None | Full TripleO integration |

---

## Troubleshooting

### Container not starting after image change

```bash
# Check Pacemaker resource status
sudo pcs status

# Check container logs
sudo podman logs $(sudo podman ps -a -q --filter name=cinder-volume)

# Check Pacemaker logs
sudo journalctl -u pacemaker -f
```

### Image tag confusion

```bash
# List all cinder-volume related images with their IDs
sudo podman images | grep -E "(cinder|IMAGE ID)"

# Inspect a specific image
sudo podman inspect <image_id>
```

### Rollback not working

```bash
# Verify backup image still exists
sudo podman images | grep pcmkoriginal

# Force container recreation
sudo pcs resource disable openstack-cinder-volume
sudo pcs resource enable openstack-cinder-volume
```

### Method 3 image build failures

```bash
# Check the tripleo container image preparation logs
tail -100 /var/log/tripleo-container-image-prepare.log

# Common issues:
# - Dockerfile syntax errors
# - Missing patch or source files
# - Registry authentication failures
# - Network connectivity issues

# Verify the modify directory exists and contains required files
ls -la /home/stack/cinder-volume-rbd/

# Test the Dockerfile manually
cd /home/stack/cinder-volume-rbd/
sudo podman build -t test-cinder:latest .
```

---

## Additional Notes

- Changes made with **Method 1** are **not persistent** across container recreations
- Changes made with **Method 2** are **persistent** as they're baked into the new image
- Pacemaker will always use the image tagged as `pcmklatest` for the cinder-volume bundle (Method 2)
- Changes made with **Method 3** are **persistent** and stored in the undercloud registry
- **Method 3** automatically applies changes to all controller nodes
- **Method 3** requires a full deployment update, making it slower but more production-ready

---

## References

- [RHOSP Container Image Management](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/)
- [TripleO Container Image Preparation](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/deployment/container_image_prepare.html)
- [Pacemaker Resource Management](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_high_availability_clusters/)
- [Podman Container Management](https://docs.podman.io/)
- [containers-prepare-parameters.yaml Example](https://paste.opendev.org/show/bgCUCTs6aIsB86vz2iuX/)
