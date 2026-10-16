# DHIS2 Instance Migration & Upgrade Guide (2.41.x to 2.42.x)

This document contains the step-by-step instructions to safely migrate and upgrade a running DHIS2 instance from version 2.41 to 2.42.

> ⚠️ **CRITICAL WARNING:** When you change the version configuration, `dhis2-server-tools` will **automatically delete and recreate the container** from scratch using a fresh Ubuntu 24.04 base image. It **does not** preserve the internal files. This makes Phase 1 (Backups) completely mandatory to prevent data loss.

---

## Considerations

### Recommended Hardware

| Component | Minimun | Required | Justification |
| :--- | :--- | :--- | :--- |
| **CPU Cores** | 32 Cores | **64 Cores** (vCPUs) | Heavy parallel processing required for resource-intensive DHIS2 analytics tables generation and high-volume background data sync loops. |
| **System Memory** | 32 GB RAM | **64 GB to 96 GB RAM** | Prevents system crashes. Under full utilization, DHIS2, the PostgreSQL cache layer, and multiple isolated environment containers easily overflow a 32GB hardware limit. |
| **Storage Capacity** | 200 GB SSD | **500 GB to 1 TB Enterprise NVMe SSD** | Database I/O is the primary bottleneck. Fast read/write performance speeds up database queries, logs tracking, and custom application file store structures. |

### Strategic Container Allocation Profiles (Resource Distribution Blueprint)
To prevent one container from starving the host memory pool and crashing other elements, the available 64GB to 96GB host allocation pool should be hard-coded into separate, isolated LXD container boundary rings using the properties layout below:

### Recommended Distribution Layout (Testing & Staging Server)

The local LXD host needs to be scaled up significantly. Because tb handles the full production-sized workload for testing, it acts as both the database and the web application layer simultaneously. Splitting tb into its functional layers right away makes its resource needs clear.

| Container / Layer | Current Allocation | Required Allocation | Justification |
| :--- | :--- | :--- | :--- |
| **tb (Web Application Layer)** | 18 GB (Shared) | **32 GB to 48 GB** | Dedicates a robust 24 GB to 32 GB Java heap to handle massive, concurrent user and analytics processing actions. |
| **tb (Database Layer)** | 18 GB (Shared) | **64 GB** | Allocates up to 24 GB of PostgreSQL shared buffers to hold the complete mirror entirely in memory. |
| **dev (Upgraded App)** | 5 GB | **32 GB** | Meeting the Tomcat 10 minimum standard and additional amout in case of intensive testing operations. |
| **gcp-integration-tb** | 3 GB | **4 GB** | Optimized to provide more space for working without any integration pipeline bottlenecks. |
| **gcp-integration-dev** | 2 GB | **3 GB** | Optimized to provide more space for working without any integration pipeline bottlenecks. |
| **postgres (dev db)** | 2 GB | **8 GB** | Ensures the development database can cache indexes and run parallel sort operations entirely in memory. |
| **Host System Overhead** | 2 GB | **8 GB** | Expanded to safely support the physical OS kernel, network routing bridges, and the default proxy/munin layers under high load. |

---

### Live Production Infrastructure Distribution (Distributed Layout)

| Server Node Tier | Required RAM | Target Internal Tuning Configurations |
| :--- | :--- | :--- |
| **Production Webapps** | **48 GB** | Allocates a 32 GB Java Virtual Machine Heap (-Xmx32g) to process large analytics queries without timeout drops, leaving 16 GB for OS overhead. |
| **Production Database** | **64 GB** | Enables up to 24 GB of shared_buffers cache memory to lock the primary data index registry entirely into RAM. |

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
git clone https://github.com/dhis2/dhis2-server-tools.git
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
sudo lxc list -c n,status,4
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
   dev dhis2_version=2.42 guest_os=24.04

   create_db=False
   
   dhis2_auto_upgrade=true 
   ```

4. Save your file changes and exit the text editor (Ctrl + O, then press Enter, then Ctrl + X).

---

## Phase 3: Automated Container Recreation

Run the automated orchestrator playbook in check mode. You must pass the custom filter plugins location path variable and attach the `--limit=dev` runtime modifier flag. This strictly locks Ansible down to the dev container namespace and guarantees your production systems remain untouched.

```bash
#Dry run
sudo ANSIBLE_FILTER_PLUGINS=filter_plugins ansible-playbook -i inventory/hosts dhis2.yml --limit=dev -e "force_recreate=true" --check

```

In case releases.json  **vars does not exist error**, we manually download the file, and try dry run again.
```bash

mkdir -p ~/dhis2-server-tools/deploy/roles/create-instance/vars

curl -s https://releases.dhis2.org/v1/versions/stable.json -o ~/dhis2-server-tools/deploy/roles/create-instance/vars/releases.json

nano ~/dhis2-server-tools/deploy/roles/create-instance/tasks/dhis2-releases.yml
```
Comment this block to skip the file automatic download:
```text
#- name: Download dhis2 releases.json file
#  delegate_to: localhost
#  run_once: true
#  become: false
#  ansible.builtin.get_url:
#    url: "https://releases.dhis2.org/v1/versions/stable.json"
#    dest: "{{ role_path }}/vars/releases.json"
#    force: true
#   mode: "0666"  # the file is public, no need to restrict permissions
#  changed_when: false

```

In case deploy-war.yml **Input version value cannot be empty error**, We wipe the content of the file replacing it, and try dry run again.


Edit file:
```bash
sudo nano ~/dhis2-server-tools/deploy/roles/create-instance/tasks/deploy-war.yml
``` 
New deploy-war.yml content:
```text
---
- name: Check current DHIS2 version
  ansible.builtin.shell: |
    if [ "{{ force_recreate | default(false) | bool }}" = "true" ]; then
      echo "0.0.0"
    elif [ -f /opt/dhis2/tomcat/webapps/{{ inventory_hostname }}/WEB-INF/lib/version.txt ]; then
      cat /opt/dhis2/tomcat/webapps/{{ inventory_hostname }}/WEB-INF/lib/version.txt | xargs
    else
      echo "0.0.0"
    fi
  register: version_results
  failed_when: false
  changed_when: false

- name: Set sanitized current dhis2 version fact
  ansible.builtin.set_fact:
    current_dhis2_version: "{{ version_results.stdout | trim if (version_results.stdout is defined and version_results.stdout | trim | length > 0) else '0.0.0' }}"

- name: Get sha256sum of dhis2.war file
  ansible.builtin.stat:
    path: /opt/dhis2/dhis2.war
    checksum_algorithm: sha256
  register: dhis2_war_stat

- ansible.builtin.debug:
    msg:
      - "Running release: {{ current_dhis2_version }}"
      - "New release: {{ patchversion_name }}"
      - set dhis2_auto_upgrade=True in your inventory file to enable automatic upgrade.
      - Prior to the upgrade, it is important to have a backup of your instance database
  when:
    - patchversion_name is defined
    - "'skipped' not in version_results.stdout | default('')"
    - current_dhis2_version is version(patchversion_name, '<')

- name: Download dhis2 war file
  ansible.builtin.command:
    cmd: wget -q -O /opt/dhis2/dhis2.war "{{ dhis2_url }}"
  when:
    - patchversion_name is defined
    - dhis2_url is defined
    - "'skipped' in version_results.stdout | default('') or current_dhis2_version is version(patchversion_name, '<') or (force_recreate | default(false) | bool)"

- name: Download dhis2.war with hard coded url
  ansible.builtin.command:
    cmd: wget -q --timeout=120 -O /opt/dhis2/dhis2.war "{{ dhis2_war_file }}"
  when:
    - dhis2_war_file is defined
    - (dhis2_war_file | regex_search('^(https?|ftp|file):\/\/')) is not none

- name: Set war file permissions
  ansible.builtin.file:
    path: /opt/dhis2/dhis2.war
    mode: "0640"
  when: >
    (patchversion_name is defined and dhis2_url is defined and
    ('skipped' in version_results.stdout | default('') or current_dhis2_version is version(patchversion_name, '<') or (force_recreate | default(false) | bool)))
    or (dhis2_war_file is defined and (dhis2_war_file | regex_search('^(https?|ftp|file):\/\/')) is not none)

- name: Copy dhis2.war file from ansible controller
  ansible.builtin.copy:
    src: "{{ dhis2_war_file }}"
    dest: /opt/dhis2/dhis2.war
    mode: "0640"
  when:
    - dhis2_war_file is defined
    - (dhis2_war_file | regex_search('^(https?|ftp|file):\/\/')) is none

- name: Clean /var/lib/tomcat{{ tomcat_version }}/webapps
  ansible.builtin.file:
    path: /var/lib/tomcat{{ tomcat_version }}/webapps/
    state: absent
  when: "'skipped' in version_results.stdout | default('') or (patchversion_name is defined and current_dhis2_version is version(patchversion_name, '<')) or (force_recreate | default(false) | bool) or dhis2_war_file is defined"

- name: Creating webapps directories
  ansible.builtin.file:
    path: /var/lib/tomcat{{ tomcat_version }}/webapps/{{ dhis2_base_path | default(inventory_hostname) | to_fixed_string }}
    mode: "0755"
    state: directory
  notify: Restart Tomcat

- name: Convert javax to jakartaee
  ansible.builtin.command:
    cmd: javax2jakarta /opt/dhis2/dhis2.war /opt/dhis2/jakarta-dhis2.war
    creates: /opt/dhis2/jakarta-dhis2.war
  when:
    - dhis2_war_file is undefined
    - normalized_dhis2_version is defined and normalized_dhis2_version.dhis2_version_major is version('2.41', '<=')
    - tomcat_version == "10"

- name: Extracting jakarta-dhis2.war
  ansible.builtin.unarchive:
    src: /opt/dhis2/jakarta-dhis2.war
    dest: /var/lib/tomcat{{ tomcat_version }}/webapps/{{ dhis2_base_path | default(inventory_hostname) | to_fixed_string }}
    remote_src: true
  when:
    - dhis2_war_file is undefined
    - normalized_dhis2_version is defined and normalized_dhis2_version.dhis2_version_major is version('2.41', '<=')
    - tomcat_version == '10'
  notify: Restart Tomcat

- name: Extracting dhis2.war file
  ansible.builtin.unarchive:
    src: /opt/dhis2/dhis2.war
    dest: /var/lib/tomcat{{ tomcat_version }}/webapps/{{ dhis2_base_path | default(inventory_hostname) | to_fixed_string }}
    remote_src: true
  when: tomcat_version != '10' or dhis2_war_file is defined and dhis2_war_file or (normalized_dhis2_version is defined and normalized_dhis2_version.dhis2_version_major is version('2.42', '>='))
  notify: Restart Tomcat

- name: Glowroot app monitoring
  ansible.builtin.include_tasks: "{{ app_monitoring }}.yml"
  when: app_monitoring is defined

- name: Flushing handlers
  ansible.builtin.meta: flush_handlers

``` 
In case glowroot.yml **Error while resolving value for 'glowroot_latest_version**, We wipe the content of the file replacing it, and try dry run again.

Edit file:
```bash
sudo nano /home/ardgzc/dhis2-server-tools/deploy/roles/create-instance/tasks/glowroot.yml
```

New glowroot.yml content:
```text
---
- name: Get latest glowroot release info
  ansible.builtin.command:
    cmd: wget -q -O - https://api.github.com/repos/glowroot/glowroot/releases/latest
  register: glowroot_release_info
  changed_when: false
  failed_when: false

- name: Set glowroot latest version
  ansible.builtin.set_fact:
    glowroot_latest_version: "{{ ((glowroot_release_info.stdout | trim if glowroot_release_info.stdout | trim | length > 0 else '{}') | from_json).tag_name | default('v0.14.7') }}"

- name: Check if glowroot directory exists
  ansible.builtin.stat:
    path: /opt/glowroot
  register: glowroot_dir

- name: Check installed glowroot version file
  ansible.builtin.stat:
    path: /opt/glowroot/.version
  register: glowroot_version_file

- name: Read installed glowroot version
  ansible.builtin.slurp:
    src: /opt/glowroot/.version
  register: glowroot_installed_version
  when: glowroot_version_file.stat.exists

- name: Set glowroot needs update fact
  ansible.builtin.set_fact:
    glowroot_needs_update: '{{ not glowroot_dir.stat.exists or (glowroot_version_file.stat.exists and (glowroot_installed_version.content | b64decode | trim) != glowroot_latest_version) }}'

- name: Download glowroot zip
  ansible.builtin.command:
    cmd: >
      wget -q -O /tmp/glowroot-dist.zip
      "https://github.com/glowroot/glowroot/releases/download/{{ glowroot_latest_version }}/glowroot-{{ glowroot_latest_version | regex_replace('^v', '') }}-dist.zip"
  when: glowroot_needs_update

- name: Extract glowroot
  ansible.builtin.unarchive:
    src: /tmp/glowroot-dist.zip
    dest: /opt/
    owner: root
    group: tomcat
    remote_src: true
    mode: '0770'
    exclude:
      - glowroot/lib/glowroot-central-collector-https-osx.jar
      - glowroot/lib/glowroot-central-collector-https-windows.jar
  when: glowroot_needs_update

- name: Save installed glowroot version
  ansible.builtin.copy:
    content: '{{ glowroot_latest_version }}'
    dest: /opt/glowroot/.version
    mode: '0644'
  when: glowroot_needs_update

- name: Clean up glowroot zip
  ansible.builtin.file:
    path: /tmp/glowroot-dist.zip
    state: absent
  when: glowroot_needs_update

- name: Find jar files in /opt/glowroot directory
  ansible.builtin.find:
    path: '/opt/glowroot/'
    recurse: true
    depth: 2
    patterns:
      - '*.jar' # search all files ending with .jar
    excludes:
      - plugin-pointcuts.jar
  register: glowroot_jar_files

- name: Change ownership and permissions of found jar files in /opt/glowroot/lib
  ansible.builtin.file:
    path: '{{ item.path }}'
    owner: root
    group: tomcat
    mode: '0640'
  loop: '{{ glowroot_jar_files.files }}'
  loop_control:
    label: '{{ item.path }}'

- name: Check if /opt/glowroot/admin.json file exists
  ansible.builtin.stat:
    path: /opt/glowroot/admin.json
  register: glowroot_admin_file_status

- name: Copy glowroot configs to the instances
  ansible.builtin.template:
    src: glowroot_admin.json
    dest: /opt/glowroot/admin.json
    owner: root
    group: tomcat
    mode: '0660'
  no_log: true
  when: not glowroot_admin_file_status.stat.exists
  notify: Restart Tomcat

- name: Configure glowroot basepath
  ansible.builtin.lineinfile:
    path: /opt/glowroot/admin.json
    regexp: '^(.*)"contextPath"'
    line: '\1"contextPath": "{{ "/glowroot" if dhis2_base_path | default(inventory_hostname) | to_fixed_string == "ROOT" else "/" + dhis2_base_path | default(inventory_hostname) | to_fixed_string + "-glowroot" }}",'
    backrefs: true
  notify: Restart Tomcat

```

Once the dry run finished with no errors we can proceed to apply the changes 
```bash

#Applying changes
sudo ANSIBLE_FILTER_PLUGINS=filter_plugins ansible-playbook -i inventory/hosts dhis2.yml --limit=dev -e "force_recreate=true"

```
*Wait for the playbook execution to finish completely with zero error blocks.*


Once the playbook execution its done with no errors, we need to download and deploy glowroot on the container:
```bash
1. Stop the broken Tomcat process loop
sudo lxc exec dev -- systemctl stop tomcat10

# 2. Create the missing destination directory footprint
sudo lxc exec dev -- mkdir -p /opt/glowroot

# 3. Download the official Glowroot release bundle directly into the container
sudo lxc exec dev -- wget https://github.com/glowroot/glowroot/releases/download/v0.14.7/glowroot-0.14.7-dist.zip -O /tmp/glowroot.zip

# 4. Install unzip inside your new container and extract the files straight to /opt/
sudo lxc exec dev -- apt update
sudo lxc exec dev -- apt install unzip -y
sudo lxc exec dev -- unzip -o /tmp/glowroot.zip -d /opt/

# 5. Fix permissions so the tomcat system engine has full read/write rights
sudo lxc exec dev -- chown -R tomcat:tomcat /opt/glowroot
sudo lxc exec dev -- chmod -R 755 /opt/glowroot

# 6. Clean up the temporary archive download file
sudo lxc exec dev -- rm /tmp/glowroot.zip
```

---

## Phase 4: Restore Environment

Now that the server tools have automatically built an empty, clean container base, you must push your original opt configurations and user data assets back inside while preventing an accidental downgrade of the software engines.

### 1. Restore System Files to the Proper Path
Push your configuration files and user-uploaded data assets back into the new container target:
```bash
# Push your backup data layers back into the container filesystem

#dhis.conf
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/dhis.conf dev/opt/dhis2/

#files
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/files dev/opt/dhis2/

#logs
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/logs dev/opt/dhis2/


# Force absolute recursive user ownership and permissions over the file maps
sudo lxc exec dev -- chown -R tomcat:tomcat /opt/dhis2
sudo lxc exec dev -- chmod -R 755 /opt/dhis2

#Restart Tomcat
sudo lxc exec dev -- systemctl restart tomcat10
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
