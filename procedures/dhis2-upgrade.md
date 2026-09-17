# DHIS2 Instance Migration & Upgrade Guide (2.41.x to 2.42.x)

This document contains the step-by-step instructions to safely migrate and upgrade a running DHIS2 instance from version

> ⚠️ **CRITICAL WARNING:** When you change the version configuration, `dhis2-server-tools` will **automatically delete and recreate the container** from scratch using a fresh Ubuntu 24.04 base image. It **does not** preserve the internal files. This makes Phase 1 (Backups) completely mandatory to prevent data loss.

---

## Phase 1: Mandatory Backups

Before running the upgrade scripts, you must pull the configurations (`dhis.conf`), files, and database schemas out of the container. If you skip this, they will be deleted when the container is recreated.

### 1. Backup the Instance `/opt` Directory
This step pulls the directory out of the `dhis` container to preserve your application settings, logs, and custom file stores on your host machine.

On your main VM terminal, run:
```bash
# Create a safe backup folder on your host VM
mkdir -p ~/dhis2-migration-backups

# Pull the entire opt ecosystem out using native LXD commands
sudo lxc file pull -r dhis/opt/ ~/dhis2-migration-backups/
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

Modifying the version tag tells the script that on the next run, recreate environment,

1. Navigate to your repository workspace directory:
   ```bash
   cd ~/dhis2-server-tools/deploy
   ```

2. Open your active configuration file:
   ```bash
   nano inventory/hosts
   ```

3. Locate your instance row under the `[instances]` section. Modify the version parameter to `2.42`. The script will automatically handle downloading the Ubuntu 24.04 base image for you:
   ```text
   # BEFORE:
   dhis dhis_version=2.41

   dhis2_auto_upgrade=false

   # AFTER:
   dhis dhis_version=2.42

   dhis2_auto_upgrade=true
   ```

4. Save changes and exit (`Ctrl + O`, then `Enter`, then `Ctrl + X`).

---

## Phase 3: Automated Container Recreation

Run the automated orchestrator script. Ansible will now wipe out the old 2.41 container, build a completely fresh **Ubuntu 24.04 container**, and install **Java 17** and **Tomcat 10**.

```bash
sudo ./deploy.sh
```
*Wait for the playbook execution to finish with zero errors.*

---

## Phase 4: Restore Environment

Now that the server tools have built a empty, clean dhis2 container, we must push your original opt files back inside.

### 1. Restore the System Files
Push your host backup copy back into the newly recreated container structure:
```bash
# Push your backup copy back into the new container
#dhis.conf
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/dhis.conf dhis/opt/dhis2/

#files
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/files dhis/opt/dhis2/

#logs
sudo lxc file push -r ~/dhis2-migration-backups/opt/dhis2/logs dhis/opt/dhis2/

```

### 2. Verify Database Status
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
sudo lxc exec dhis -- systemctl restart tomcat10

# Follow the updated Tomcat 10 log trail to watch the boot sequence
sudo lxc exec dhis -- tail -f /var/log/tomcat10/catalina.out
```
Once the log reads `Server startup in XXX ms` with no database connection errors, your migration to **DHIS2 2.42.x running on Tomcat 10** is complete!
