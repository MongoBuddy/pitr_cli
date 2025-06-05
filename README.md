# **MongoDB PITR CLI Utility (pitr\_cli.py)**

## **1\. Overview**

`pitr_cli.py` is a command-line Python utility designed to perform Point-in-Time Recovery (PITR) for MongoDB. It utilizes oplog entries collected by a separate [oplog collector](https://github.com/MongoBuddy/oplog_collector) script (assumed to be stored in a dedicated Oplog Data Store MongoDB). This tool allows users to check the available time range for recovery and to apply oplogs to a restored MongoDB instance to bring it to a specific point in time.

This script is intended to be used in conjunction with a base backup/snapshot restoration (e.g., LVM snapshot, EBS snapshot, or `mongodump`).

## **2\. Features**

* **Check Recoverable Time Range**: Determines the oldest and newest oplog timestamps available in the Oplog Data Store for a specific cluster.  
* **Apply Oplogs**: Applies oplog entries from the Oplog Data Store to a target MongoDB instance for a specified time range.  
  * Handles DML operations (insert, update, delete) in batches.  
  * Handles DDL/Command operations (drop, dropDatabase, rename, create, collMod) individually via applyOps.  
  * Handles `createIndexes` and `deleteIndexes` via direct database commands.  
  * Attempts to automatically create collections if an `insert` operation targets a non-existent namespace (due to `NamespaceNotFound` error on `applyOps`).  
  * Displays progress percentage during the oplog application process.  
* **Configurable Logging**: Log level can be controlled via an environment variable, and logs are rotated.

## **3. Prerequisites**

* Python 3.7+  
* `pymongo` Python library (install via `pip install pymongo`)  
* An Oplog Data Store MongoDB instance populated by `oplog_collector.py` for the source cluster.  
* A base backup (e.g., volume snapshot, `mongodump`) of the source MongoDB instance that you intend to restore.

## **4\. Setup**

1. Ensure `pitr_cli.py` is located in your project's `scripts/` directory.  
2. Install the required Python library:
```
   pip install pymongo
```
## **5\. Environment Variables**

* `PITR_CLI_LOG_LEVEL`: Sets the logging level for the script.  
  (Default: `INFO`. Options: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`)  
  Example: `export PITR_CLI_LOG_LEVEL=INFO`  
* `PITR_APPLYOPS_BATCH_SIZE`: Size of batches for DML operations during `applyOps`.  
  (Default: `100`)  

## **6\. Usage**

The script is run from the command line and takes a command argument (`get_range` or `apply`) along with other necessary parameters.

### **6.1. Check Recoverable Time Range (`get_range`)**

This command shows the time window for which oplogs are available for the specified cluster.

**Command Syntax:**
```
python scripts/pitr_cli.py <cluster_name> <oplog_store_uri> get_range
```
**Parameters:**

* `<cluster_name>`: The name of the source cluster (as used by `oplog_collector.py` to store its oplogs).  
* `<oplog_store_uri>`: The MongoDB connection URI for the Oplog Data Store.

**Example:**
```
python scripts/pitr_cli.py myProdCluster mongodb://localhost:27017/oplogArchiveDB get_range
```
Output will be similar to:
```
Recoverable Time Range (UTC):  
  From: 2025-06-01T01:00:00Z  
  To  : 2025-06-03T14:30:00Z
```
### **6.2. Apply Oplogs for PITR (`apply`)**

This command applies oplogs to a target MongoDB instance to recover it to a specific point in time. This is typically done *after* restoring a base backup/snapshot.

**Command Syntax:**
```
python scripts/pitr_cli.py <cluster_name> <oplog_store_uri> apply \  
    --from_time <start_iso_utc_time> \  
    --to_time <target_iso_utc_time> \  
    --target_url <target_mongodb_uri>
```
**Parameters:**

* `<cluster_name>`: The name of the source cluster.  
* `<oplog_store_uri>`: The MongoDB connection URI for the Oplog Data Store.  
* `--from_time` (`-f`) `YYYY-MM-DDTHH:MM:SSZ`: The UTC timestamp from which to start applying oplogs (exclusive). This should typically be a time *just before or at* the point your base backup/snapshot was taken or consistent.  
* `--to_time` (`-t`) `YYYY-MM-DDTHH:MM:SSZ`: The UTC timestamp to recover to (inclusive). Oplogs up to and including this time will be applied.  
* `--target_url`: The MongoDB connection URI of the target instance where the base backup has been restored and to which oplogs will be applied.

**Example Workflow with Volume Snapshot:**

Let's say you take daily LVM/EBS volume snapshots of your MongoDB data volume at 02:00 AM UTC.  
You need to recover your database to 04:35 AM UTC on June 3rd, 2025\.

1. **Restore Snapshot**:  
   * Restore the MongoDB data volume from the snapshot taken at `2025-06-03T02:00:00Z` to a new or clean MongoDB instance.  
   * Start this restored MongoDB instance. It will be in the state it was at approximately 02:00 AM.  
2. **Apply Oplogs using `pitr_cli.py`**:  
   * The `oplog_collector.py` should have been continuously collecting oplogs into your Oplog Data Store.  
   * To roll forward from the snapshot time to the desired recovery point:
  ```
     python scripts/pitr_cli.py myProdCluster mongodb://oplog_store_host/oplogArchiveDB apply \  
         --from_time "2025-06-03T01:59:55Z" \  
         --to_time "2025-06-03T04:35:00Z" \  
         --target_url "mongodb://user:pass@restored_mongo_host:27017/admin"
```
     
The script will then connect to the Oplog Data Store, fetch oplogs between `--from_time` (exclusive) and `--to_time` (inclusive) for `myProdCluster`, and apply them to the MongoDB instance at `restored_mongo_host`.

* Note: `--from_time` is set slightly before the snapshot time (e.g., 5 seconds before 02:00:00Z) to ensure all relevant operations around the snapshot point are included. The exact "from" time might need adjustment based on your snapshotting procedure and `fsyncLock` timing.



## **7\. Important Considerations**

* **Data Consistency of Base Backup/Snapshot**: For a successful PITR, the base backup or snapshot must be taken from a consistent state of the MongoDB instance. For filesystem snapshots (LVM, EBS), it's crucial to run `db.fsyncLock()` on the source MongoDB instance before taking the snapshot and `db.fsyncUnlock()` after the snapshot is complete.  
* **Accurate Timestamps**: Ensure the `--from_time` and `--to_time` arguments are accurate UTC timestamps. The `--from_time` should correspond closely to the consistent state time of your restored base backup.  
* **Oplog Availability**: This tool relies on the continuous and complete collection of oplogs by `oplog_collector.py` into the Oplog Data Store. Any gaps in oplog collection will result in an incomplete recovery for that period.  
* **Target Instance**: The target MongoDB instance specified by `--target_url` should be the instance where the base backup/snapshot has already been restored.
