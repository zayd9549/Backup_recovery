# **Oracle 19c Backup and Recovery: Practical Hands-On Guide**

This document is designed to provide practical steps for **Oracle 19c Backup and Recovery**, including handling common scenarios such as **datafile loss**. It also provides a detailed explanation of **consistent** vs. **non-consistent backups**, as well as **full recovery steps** to restore and recover data files.

---

## **Pre-Requisites:**

1. **Oracle 19c Installed**: Ensure Oracle 19c is installed and configured correctly, and a test database is created.
2. **Database Accessible**: Ensure access to SQL\*Plus or any other client with **`sysdba`** privileges to manage backups and recovery.
3. **Backup Location**: Ensure there is a designated location for backups (e.g., an external disk, directory, or cloud storage).
4. **Backup Tools**: Familiarize yourself with Oracle SQL\*Plus commands, `cp` (copy) for OS-level file backup, and other utilities like **RMAN** (if applicable).

---

## **1. Cold Backup (Offline Backup)**

A **Cold Backup** involves shutting down the Oracle database to ensure that no transactions are in progress while performing the backup. This ensures consistency in the backup.

### **Steps for Cold Backup:**

1. **Shutdown the Database:**

   * To ensure no active transactions are occurring, shut down the database:

   ```sql
   SQL> SHUTDOWN IMMEDIATE;
   ```

2. **Backup the Files:**

   * Use the OS-level `cp` command to copy the necessary database files (datafiles, control files, redo logs) to a backup location:

   ```bash
   $ cp -v /u01/app/oracle/oradata/*.dbf /backup/location/
   $ cp -v /u01/app/oracle/dbs/*.ctl /backup/location/
   $ cp -v /u01/app/oracle/redo/*.log /backup/location/
   ```

3. **Restart the Database:**

   * After backing up the files, restart the database:

   ```sql
   SQL> STARTUP;
   ```

4. **Archive the Redo Logs (Optional):**

   * It's recommended to archive redo logs before shutting down to ensure all changes are captured:

   ```sql
   SQL> ALTER SYSTEM ARCHIVE LOG CURRENT;
   ```

---

### **Key Takeaways:**

* **Cold Backup** requires downtime but guarantees that the database is in a consistent state.
* It's the most straightforward backup method but isn't suitable for environments that require 24/7 availability.

---

## **2. Hot Backup (Online Backup)**

A **Hot Backup** allows you to perform a backup while the database is running and accessible to users. It uses **archive logs** to ensure consistency while the database is live.

### **Steps for Hot Backup:**

1. **Perform a Log Switch:**

   * To ensure no data is lost, switch the redo log to a new file:

   ```sql
   SQL> ALTER SYSTEM SWITCH LOGFILE;
   ```

2. **Put the Database in Backup Mode:**

   * Enable backup mode for the database:

   ```sql
   SQL> ALTER DATABASE BEGIN BACKUP;
   ```

3. **Copy Data Files:**

   * Use the OS-level `cp` command to copy all datafiles to the backup location:

   ```bash
   $ cp -v /u01/app/oracle/oradata/*.dbf /backup/location/
   ```

4. **End Backup Mode:**

   * Once the copy is complete, disable backup mode:

   ```sql
   SQL> ALTER DATABASE END BACKUP;
   ```

5. **Backup Control Files:**

   * Backup the control files to ensure all metadata is captured:

   ```sql
   SQL> ALTER DATABASE BACKUP CONTROLFILE TO '/backup/location/control.bkp';
   ```

6. **Backup Archived Redo Logs:**

   * Ensure the archive redo logs are backed up as well:

   ```bash
   $ cp -v /u01/app/oracle/archivelogs/*.log /backup/location/
   ```

---

### **Key Takeaways:**

* **Hot Backup** ensures the database remains available during the backup process.
* Archive logs are crucial for maintaining consistency during hot backups, as transactions may be ongoing.

---

## **3. Recovery Scenarios:**

### **Scenario 1: Loss of Non-System Datafile (User Datafile)**

When a **non-system datafile** (such as `user01.dbf`) is lost, recovery involves restoring the missing datafile and applying archived redo logs.

#### **Steps for Recovery:**

1. **Offline the Lost Datafile:**

   * Take the lost datafile offline to prevent errors during recovery:

   ```sql
   SQL> ALTER DATABASE DATAFILE '/path/to/datafile/user01.dbf' OFFLINE;
   ```

2. **Restore the Datafile:**

   * Restore the lost datafile from the backup:

   ```bash
   $ cp /backup/location/user01.dbf /u01/app/oracle/oradata/prod/
   ```

3. **Recover the Datafile:**

   * Recover the datafile using archived redo logs:

   ```sql
   SQL> RECOVER DATAFILE '/u01/app/oracle/oradata/prod/user01.dbf';
   ```

4. **Bring the Datafile Online:**

   * Once recovery is complete, bring the datafile back online:

   ```sql
   SQL> ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/prod/user01.dbf' ONLINE;
   ```

---

### **Key Takeaways:**

* **Datafile recovery** involves taking the datafile offline, restoring from backup, recovering with redo logs, and bringing the datafile back online.
* Ensure **archive logs** are available for a successful recovery.

---

### **Scenario 2: Loss of Control File**

If a **control file** is lost, the database needs to be restored from the last known good backup.

#### **Steps for Recovery:**

1. **Shutdown the Database:**

   * Shutdown the database in **abort mode**:

   ```sql
   SQL> SHUTDOWN ABORT;
   ```

2. **Mount the Database:**

   * Start the database in **mount mode** to restore the control files:

   ```sql
   SQL> STARTUP MOUNT;
   ```

3. **Restore the Control Files:**

   * Restore the control files from backup:

   ```bash
   $ cp /backup/location/bkp.ctl /u01/app/oracle/dbs/control01.ctl
   $ cp /backup/location/bkp.ctl /u01/app/oracle/dbs/control02.ctl
   ```

4. **Recover the Database:**

   * Recover the database using the backup control file:

   ```sql
   SQL> RECOVER DATABASE USING BACKUP CONTROLFILE UNTIL CANCEL;
   ```

5. **Open the Database with RESETLOGS:**

   * After recovery, open the database with `RESETLOGS`:

   ```sql
   SQL> ALTER DATABASE OPEN RESETLOGS;
   ```

---

### **Key Takeaways:**

* **Control file loss** requires restoring control files from backup and performing recovery.
* **RESETLOGS** is used to open the database after control file recovery to ensure consistency.

---

### **Scenario 3: Loss of All Datafiles**

In the worst-case scenario where **all datafiles** are lost, you need to restore and recover all datafiles from backup.

#### **Steps for Recovery:**

1. **Shutdown the Database in Abort Mode:**

   ```sql
   SQL> SHUTDOWN ABORT;
   ```

2. **Start the Database in Mount Mode:**

   ```sql
   SQL> STARTUP MOUNT;
   ```

3. **Restore All Datafiles:**

   * Restore all datafiles from backup:

   ```bash
   $ cp /backup/location/*.dbf /u01/app/oracle/oradata/prod/
   ```

4. **Recover the Database:**

   * Recover the database to apply all changes:

   ```sql
   SQL> RECOVER DATABASE;
   ```

5. **Open the Database:**

   * After recovery, open the database:

   ```sql
   SQL> ALTER DATABASE OPEN;
   ```

---

### **Key Takeaways:**

* **Loss of all datafiles** is a critical situation requiring the restoration of **all datafiles** and **database recovery**.
* Use **archive logs** to ensure all committed transactions are applied during recovery.

---

### **4. Incomplete Recovery (When Archive Logs are Missing)**

If **archive logs** are missing, Oracle allows for **incomplete recovery**, where the database is recovered up to a specific point in time.

#### **Steps for Incomplete Recovery:**

1. **Restore All Datafiles:**

   ```bash
   $ cp /backup/location/*.dbf /u01/app/oracle/oradata/prod/
   ```

2. **Start the Database in Mount Mode:**

   ```sql
   SQL> STARTUP MOUNT;
   ```

3. **Recover the Database Until a Specific Time/Sequence:**

   * For time-based recovery:

   ```sql
   SQL> RECOVER DATABASE UNTIL TIME '2022-07-17:14:40:45';
   ```

   * For sequence-based recovery:

   ```sql
   SQL> RECOVER DATABASE UNTIL SEQUENCE 10 THREAD 1;
   ```

4. **Open the Database with RESETLOGS:**

   ```sql
   SQL> ALTER DATABASE OPEN RESETLOGS;
   ```

---

### **Key Takeaways:**

* **Incomplete recovery** is useful when archive logs are missing, but it requires careful use of `UNTIL TIME`, `UNTIL SEQUENCE`, or `UNTIL CANCEL` for point-in-time recovery.
* **RESETLOGS** must be used to complete the recovery after incomplete recovery.

---

## **Consistent vs Non-Consistent Backups**

Understanding the difference between **consistent** and **non-consistent backups** is essential for determining recovery strategies.

### **Consistent Backup:**

* A **consistent backup** occurs when all database files are in sync, which usually happens when the database is either **shut down** or placed in **backup mode** (e.g., hot backup mode).
* **Example**: A **Cold Backup** is a **consistent backup** since the database is in a consistent state when shut down.

### **Non-Consistent Backup:**

* A **non-consistent backup** occurs when the database is running, and transactions are actively being processed. In such cases, the backup may contain data in an inconsistent state.
* **Example**: A **Hot Backup** is a **non-consistent backup** as the database continues processing transactions while the backup is happening.
* **Recovery for non-consistent backups** requires applying **archived redo logs** to make the database consistent.

---

### **Key Takeaways:**

* **Consistent backups** guarantee database consistency and don’t require further recovery operations.
* **Non-consistent backups** need **redo log recovery** to apply changes and make the database consistent.

---
