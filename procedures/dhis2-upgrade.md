# DHIS2 Instance Migration & Upgrade Guide (2.41.x to 2.42.x)

This document contains the step-by-step instructions to safely migrate and upgrade a running DHIS2 instance from version 2.41 to 2.42.

> ⚠️ **CRITICAL WARNING:** When you change the version configuration, `dhis2-server-tools` will **automatically delete and recreate the container** from scratch using a fresh Ubuntu 24.04 base image. It **does not** preserve the internal files. This makes Phase 1 (Backups) completely mandatory to prevent data loss.

---

## Considerations

### Recommended Hardware

| Component | Required | Optimal | Justification |
| :--- | :--- | :--- | :--- |
| **CPU Cores** | 32 Cores | **32 Cores** (vCPUs) | Heavy parallel processing required for resource-intensive DHIS2 analytics tables generation and high-volume background data sync loops. |
| **System Memory** | 32 GB RAM | **64 GB to 96 GB RAM** | Prevents system crashes. Under full utilization, DHIS2, the PostgreSQL cache layer, and multiple isolated environment containers easily overflow a 32GB hardware limit. |
| **Storage Capacity** | 200 GB SSD | **500 GB to 1 TB Enterprise NVMe SSD** | Database I/O is the primary bottleneck. Fast read/write performance speeds up database queries, logs tracking, and custom application file store structures. |

### Strategic Container Allocation Profiles (Resource Distribution Blueprint)
To prevent one container from starving the host memory pool and crashing other elements, the available 64GB to 96GB host allocation pool should be hard-coded into separate, isolated LXD container boundary rings using the properties layout below:

| Container | Current Allocation | New Allocation | Justification |
| :--- | :--- | :--- | :--- |
| **tb** | 18GB | 32GB | Plenty of headroom to handle large historical case evaluations. |
| **dev** | 5 GB | 12GB | Meeting the Tomcat 10 minimum standard and additional amout in case of intensive testing operations. |
| **gpc-integration-tb** | 3GB | 4GB | Optimized to provide more space for working without any bottleneck. |
| **gpc-integration-dev** | 2GB | 3GB | Optimized to provide more space for working without any bottleneck. |
| **posgres (dev db)** | 2GB | 8GB | Ensures the database can cache indexes and run parallel sort operations in memory. |
| **Host System Overhead** | 2GB | 5GB | Retained for the physical OS kernel, the network bridges, and the proxy/munin containers.
---

## How to Audit Live Resource Allocation and Infrastructure Budgets

If the host environment uses a resource distributor, allocation framework, or manual control profiles, the following execution protocols must be run directly on the host machine to extract exactly how many hardware resources are assigned to each container block.

### 1. View Configured LXD Container Hard Constraints
To inspect if explicit memory limits or CPU execution counts have been locked onto any container, execute the following configuration inspection commands:
```bash
# Display the exact resource restrictions assigned to the core app container
sudo lxc config show dev

# Inspect the system profile rules enforcing limits across the database layer
sudo lxc config show postgres
```
*Look specifically for lines defining limits.memory or limits.cpu. If these entries are completely missing from the metadata dictionary map, it implies the container has zero limits and is running unrestricted over the host resource pools.*

### 2. Audit Global Profiling Architecture
To inspect the master templates assigned to groups of containers by the original automation script, run:
```bash
# List all active operational profile templates
sudo lxc profile list

# Inspect the explicit layout limits of the master DHIS2 server profile template
sudo lxc profile show dhis2-profile
```

### 3. Extract Real-Time Cluster Node Topography
If the host server is tied into a multi-node cluster architecture that distributes virtual container files across multiple physical machines, view the master ledger status directly:
```bash
sudo lxc cluster list
```

---

## Workspace Setup: Locating or Recloning the Infrastructure Repository

Before checking parameters, run this sequence on the host server to locate original setup files or pull down a fresh working copy to coordinate the upgrade.

### 1. Search for Existing dhis2-server-tools Configuration Maps
Execute a root-level search to locate any existing repository instances:
```bash
sudo find / -type d -name "dhis2-server-tools" 2>/dev/null
```

### 2. Handle Workspace Strategy Based on Search Results

#### Scenario A: The Folder Exists
If the search returns an active directory path (for example, `/home/ubuntu/dhis2-server-tools`), navigate directly into its deployment workspace folder and skip the git clone step:
```bash
cd /path/to/discovered/dhis2-server-tools/deploy
```

#### Scenario B: The Folder Does Not Exist
If the search command returns no output, the original setup was likely handled remotely or deleted. Pull down a fresh copy of the framework to manage the local environment:
```bash
# Clone the repository onto your local host space
git clone git@github.com:dhis2/dhis2-server-tools.git
cd dhis2-server-tools/deploy

# Generate a clean inventory file from the repository blueprint template
cp inventory/hosts.template inventory/hosts
```

---

## Infrastructure Discovery: Extracting Properties for the Inventory File

Run these commands on the live host server to discover the exact networking variables, subnets, and host parameters required to populate or verify your `inventory/hosts` configuration file.

### 1. Identify the Internal LXD Container Network Bridge
To identify what network bridge is connecting your containers and find its explicit subnet address range, execute:
```bash
sudo lxc network list
```
*   **What to look for:** Locate the network device name that is MANAGED and in use (typically lxdbr1 or lxdbr0). Note its name and the IPV4 GATEWAY/SUBNET string (e.g., 172.19.2.1/24).

### 2. Extract Exact Network Mapping Parameters
To pull the exact, detailed configuration details of that specific bridge interface, run:
```bash
sudo lxc network show lxdbr1
```
*(Replace lxdbr1 with the exact managed bridge name discovered in the previous step).*

*   **Variables to extract:**
    *   `ipv4.address`: This string goes directly into your hosts file as `lxd_network=` (e.g., 172.19.2.1/24).
    *   The bridge name itself goes into your hosts file as `lxd_bridge_interface=` (e.g., lxdbr1).

### 3. Discover Active Container Instance Names
To verify the exact labels assigned to the application container layers (ensuring you target dev and freeze tb correctly), execute:
```bash
sudo lxc list -c n,status,ipv4
```
*   **What to look for:** Copy down the exact text listed in the NAME column for your development instance (dev) and your testing/production instance (tb). These labels must match your [instances] block headers identically.

---

### Reconstructed inventory/hosts Reference Template

Based on the discovered system metrics, populate your local configuration file (inventory/hosts) using this exact isolated grouping structure. Placing tb inside a separate frozen group blocks Ansible from applying changes to it during your targeted deployment run:

```ini
[all:vars]
# 1. System Access Parameters
ansible_connection=lxd
timezone=Asia/Tashkent
email=admin@yourdomain.com

# 2. Network Properties (Extracted from Step 2)
lxd_bridge_interface=lxdbr1
lxd_network=172.19.2.1/24

[instances]
# 3. Target Allocations (Extracted from Step 3)
dev dhis_version=2.42 guest_os=24.04

[production_frozen]
# Hard-isolated block. Ansible will skip modifying this target group
tb  dhis_version=2.41 guest_os=22.04
```
---
### Reconstructed inventory/hosts Reference Template

Based on the discovered system metrics and active container footprints, configuration file (inventory/hosts) should look like this:

```ini
[all:vars]
# 1. System Access Parameters
ansible_connection=lxd
timezone=Asia/Tashkent
email=admin@yourdomain.com

# 2. Network Properties (Extracted from Step 2)
lxd_bridge_interface=lxdbr1
lxd_network=172.19.2.1/24

[instances]
# 3. Active Target: Upgrading dev to version 2.42 on Tomcat 10
dev dhis_version=2.42 guest_os=24.04

[production_frozen]
# 4. Protected Environment: Keep the Testbed (tb) safely on 2.41
tb dhis_version=2.41 guest_os=22.04

[custom_plugins:children]
# 5. Supplementary Clusters: Keeping integrations unmanaged during this run
gcp_integrations

[gcp_integrations]
# 6. Specialized Civil Reg Plugins (Listed to prevent configuration drift)
gcp-integration-dev
gcp-integration-tb
```

---

## Phase 1: Mandatory Backups

Before running the upgrade scripts, you must pull the configurations (dhis.conf), files, and database schemas out of the container. If you skip this, they will be deleted when the container is recreated.

### 1. Backup the Instance /opt Directory
This step pulls the directory out of the dev container to preserve your application settings, logs, and custom file stores on your host machine.

On your main VM terminal, run:
```bash
# Create a safe backup folder on your host VM
mkdir -p ~/dhis2-migration-backups

# Pull the entire opt ecosystem out using native LXD commands
sudo lxc file pull -r dev/opt/ ~/dhis2-migration-backups/
```

### 2. Backup the PostgreSQL Database
Generate a manual database dump file snapshot directly from the container database cluster just in case.
```bash
# Enter the postgres container and generate a compressed sql dump
sudo lxc exec postgres -- pg_dump -U postgres -F c -b -v -f /tmp/dhis2_pre_upgrade.dump dhis2

# Pull the generated dump file out onto your host machine
sudo lxc file pull postgres/tmp/dhis2_pre_upgrade.dump ~/dhis2-migration-backups/
```

---

---

## Phase 2: Modify Version Target

Modifying the version tag tells the script that on the next execution, it must wipe out the outdated instance framework and allocate the new infrastructure context.

1. Ensure you are inside the active deployment directory:
   ```bash
   cd ~/dhis2-server-tools/deploy
   ```

2. Open your active configuration file:
   ```bash
   nano inventory/hosts
   ```

3. Locate your dev row under the [instances] section. Confirm it targets the correct upgrade version values and appropriate guest operating system profile:
   ```text
   dev dhis_version=2.42 guest_os=24.04
   ```

4. Save your file changes and exit the text editor (Ctrl + O, then press Enter, then Ctrl + X).

---

## Phase 3: Automated Container Recreation

Run the automated orchestrator playbook. You must pass the custom filter plugins location path variable and attach the `--limit=dev` runtime modifier flag. This strictly locks Ansible down to the dev container namespace and guarantees your production systems remain untouched.

```bash
ANSIBLE_FILTER_PLUGINS=filter_plugins ansible-playbook -i inventory/hosts dhis2.yml --limit=dev -e "force_recreate=true"

```
*Wait for the playbook execution to finish completely with zero error blocks.*

---

## Phase 4: Restore Environment

Now that the server tools have automatically built an empty, clean container base, you must push your original opt configurations and user data assets back inside while preventing an accidental downgrade of the software engines.

### 1. Clean Up the Old Core Executables Before Transfer
To ensure you do not copy your old 2.41 web archives over the newly installed 2.42 Tomcat 10 files, wipe the old war file from the local backup directory:
```bash
rm -rf ~/dhis2-migration-backups/opt/dhis2/tomcat/webapps/*
```

### 2. Restore System Files to the Proper Path
Push your configuration files and user-uploaded data assets back into the new container target:
```bash
# Push your backup data layers back into the container filesystem
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2 dev/opt/

# Force absolute recursive user ownership and permissions over the file maps
sudo lxc exec dev -- chown -R tomcat:tomcat /opt/dhis2
sudo lxc exec dev -- chmod -R 755 /opt/dhis2

# Clear the temporary application installation folder to ensure clean custom app loads
sudo lxc exec dev -- rm -rf /opt/dhis2/apps/*
```

### 3. Verify Database Status
If the database engine container configuration remained untouched and your data is accessible, skip this step. If the script wiped the database, recreate the empty shell and feed your dump file back into it:
```bash
# Recreate an empty database framework shell
sudo lxc exec postgres -- dropdb -U postgres dhis2 2>/dev/null || true
sudo lxc exec postgres -- createdb -U postgres -O dhis dhis2

# Push your backup dump file back into the database container
sudo lxc file push ~/dhis2-migration-backups/dhis2_pre_upgrade.dump postgres/tmp/

# Run the restore process using pg_restore
sudo lxc exec postgres -- pg_restore -U postgres -d dhis2 -v /tmp/dhis2_pre_upgrade.dump
```

---

## Phase 5: Reboot and Verify

Restart the application service so the updated 2.42 core reads the restored configuration paths and processes the automatic background database upgrades.

```bash
# Restart the new Tomcat 10 server application inside the container
sudo lxc exec dev -- systemctl restart tomcat10

# Follow the updated Tomcat 10 log trail to watch the boot sequence
sudo lxc exec dev -- tail -f /var/log/tomcat10/catalina.out
```
Once the log reads `Server startup in XXX ms` with no database connection errors, your migration to **DHIS2 2.42.x running on Tomcat 10** is complete!
