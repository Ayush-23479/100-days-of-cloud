# 🛠️ Day 25: Expanding and Managing Disk Storage

### 📋 Scenario & Goal
The Nautilus DevOps Team required an upgrade to the storage subsystem of the virtual machine `nautilus-vm` to handle expanding logs and application assets. The task had two clear objectives:
1. Expand the primary virtual OS disk from 32Gi to 64Gi.
2. Create and attach a new 64Gi Standard HDD data disk named `nautilus-disk` and mount it persistent-state inside the Linux filesystem under `/mnt/nautilus-disk`.

### 💡 Core Engineering Lessons Learned
* **Azure Linux Device Mapping Convention:** Understood that Azure reserves `/dev/sdb` specifically for temporary scratch and swap space managed by the local Azure Linux Agent (`waagent`). Direct manual attempts to format `/dev/sdb` will fail with errors stating the resource is busy or in use. Newly attached user data disks map to subsequent block letters, starting at `/dev/sdc`.
* **Automated Host-Side Resizing:** Modern cloud-optimized Linux images run partition-stretching processes (`growpart` / `resize2fs`) automatically during the boot phase once they detect an increase in physical disk provisioning.
* **Persistent Mounting Protocol:** Standardized mounting configurations using unique device UUIDs inside `/etc/fstab` to ensure the instance does not enter a boot panic state if block letters shift during maintenance windows.

---

## 🔧 Implementation Methodologies

### Option 1: Programmatic Infrastructure Execution (Azure CLI)

1. **Query Session Context:**
   Obtain the dynamic target Resource Group name for the sandbox session:
   ```bash
   az group list --query '[].name' --output table | grep 'kml'

    Resize the OS Drive Resource:
    Stop the instance, resize the backend disk volume, and reboot:
    Bash

    # Deallocate compute resource to release file lock
    az vm deallocate --resource-group <YOUR_RESOURCE_GROUP> --name nautilus-vm

    # Query the unique name of the underlying OS Disk
    OS_DISK_NAME=$(az vm show --resource-group <YOUR_RESOURCE_GROUP> --name nautilus-vm --query "storageProfile.osDisk.name" -o tsv)

    # Expand the physical drive capacity to 64GB
    az disk update --resource-group <YOUR_RESOURCE_GROUP> --name $OS_DISK_NAME --size-gb 64

    # Return the VM to an active running state
    az vm start --resource-group <YOUR_RESOURCE_GROUP> --name nautilus-vm

    Provision and Attach the HDD Storage Block:
    Create and hot-attach the target standard storage block volume:
    Bash

    az vm disk attach \
      --resource-group <YOUR_RESOURCE_GROUP> \
      --vm-name nautilus-vm \
      --name nautilus-disk \
      --new \
      --size-gb 64 \
      --sku Standard_LRS

Option 2: Visual Control Plane Management (Azure Portal)

    Expand OS Storage Block:

        Navigate to Virtual Machines ➡️ nautilus-vm.

        Click Stop on the top command ribbon and wait for Stopped (deallocated) status.

        On the left sidebar, click Disks. Click directly on the active OS disk name string.

        On the Disk Blade sidebar, navigate to Size + performance. Scale the value to 64 GiB and click Save.

        Return to nautilus-vm overview page and click Start.

    Map the Secondary Volume (nautilus-disk):

        Go back to the VM's Disks page.

        Under the Data disks frame, select Create and attach a new disk.

        Set Disk name as nautilus-disk, change the Storage type strictly to Standard HDD, and set the capacity to 64 GiB.

        Click Save to complete the block attachment.

🐧 Operating System Level Provisioning (SSH Console)

Once the VM is running, connect securely from the landing host to format and mount the secondary storage volume correctly:
Bash

ssh azureuser@<YOUR_VM_PUBLIC_IP>

    Verify Block Storage Topography:
    Run the list block devices tool:
    Bash

    lsblk

    Expected Output structure:

        sda (64G) ➡️ Automated filesystem expansion.

        sdb (4G-8G) ➡️ Reserved Azure temporary scratch drive. DO NOT FORMAT.

        sdc (64G) ➡️ The target unformatted, raw data volume.

    Format & Persistent Mounting Sequence:
    Format, configure the static directory pointer, and apply persistence:
    Bash

    # Format the correct drive path (/dev/sdc)
    sudo mkfs -t ext4 /dev/sdc

    # Anchor a new directory within the filesystem structure
    sudo mkdir -p /mnt/nautilus-disk

    # Perform the initial mount operation
    sudo mount /dev/sdc /mnt/nautilus-disk

    # Capture the unique UUID of /dev/sdc and write persistence to fstab
    DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdc)
    echo "UUID=$DISK_UUID /mnt/nautilus-disk ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

    # Execute dry-run to ensure /etc/fstab compiles cleanly
    sudo mount -a

    Verify Mount Integrity:
    Ensure the device maps to the correct target block directory:
    Bash

    df -h | grep nautilus-disk


