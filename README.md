### **MongoDB PITR CLI Utility (pitr\_cli.py)**

## **1\. Overview**

`pitr_cli` is a command-line utility designed for two primary purposes: Point-in-Time Recovery (PITR) and Workload Replay for MongoDB. It utilizes oplog entries collected by a separate Oplog Collector system (like the one available at https://github.com/MongoBuddy/oplog_collector), which are assumed to be stored in a dedicated Oplog Data Store MongoDB.

This tool allows users to:

* For PITR: Apply oplogs to a restored base backup (e.g., volume snapshot, mongodump) to bring a database to a specific moment in time.  
* For Workload Replay: Replay a captured production workload on a different environment (e.g., staging, development) for performance testing, index validation, or bug analysis.  
* Check the available time range for recovery or replay based on the collected oplogs.

## **2\. Licensing & Availability**
* License Mechanism: To use this tool, a valid `license.key` file is required. This license file is valid for approximately one month from the date of issue. To continue using the tool, a new license file must be obtained.  
* Platform: Currently, a pre-compiled binary for macOS (Apple Silicon) is provided.  

To inquire about obtaining a license, please contact us at: MongoBuddyNow@gmail.com

## **3\. Prerequisites**

* A compiled binary for your OS (currently macOS is available).  
* A valid `license.key` file.  
* An Oplog Data Store MongoDB instance populated by an Oplog Collector.  
* A base backup (e.g., `volume snapshot`, `mongodump`) of the source MongoDB instance that you intend to restore or use as a baseline for replay.

## **4\. Setup**

1. Place the compiled `pitr_cli` executable file in your desired directory.  
2. Place your provided `license.key` file in the same directory as the `pitr_cli` executable.  
3. Ensure the executable has the necessary permissions:
   ```
   chmod +x /path/to/your/pitr_cli
   ```

## **5\. Usage**

The script is run from the command line.

### **5.1. Check Recoverable Time Range (`get_range`)**

This command shows the time window for which oplogs are available for the specified cluster.

Command Syntax:
```
./pitr_cli <cluster_name> <oplog_store_uri> get_range
```
Parameters:

* `<cluster_name>`: The name of the source cluster (as used by the Oplog Collector to store its oplogs).  
* `<oplog_store_uri>`: The MongoDB connection URI for the Oplog Data Store.

Example:
```
./pitr_cli rs0 "mongodb://user:password@localhost:40000/oplog_prod_db?replicaSet=oplog" get_range
```

### **5.2. Apply Oplogs for PITR or Workload Replay (`apply`)**

This command applies oplogs to a target MongoDB instance.

Command Syntax:
```
./pitr_cli <cluster_name> <oplog_store_uri> apply \  
    --from_time <start_iso_utc_time> \  
    --to_time <target_iso_utc_time> \  
    --target_url <target_mongodb_uri>
```
Parameters:

* `<cluster_name>`: The name of the source cluster.  
* `<oplog_store_uri>`: The MongoDB connection URI for the Oplog Data Store.  
* `--from_time (-f) YYYY-MM-DDTHH:MM:SSZ`: The UTC timestamp from which to start applying oplogs (exclusive). For PITR, this is typically the snapshot time. For workload replay, this is the start of the desired workload window.  
* `--to_time (-t) YYYY-MM-DDTHH:MM:SSZ`: The UTC timestamp to recover or replay to (inclusive).  
* `--target_url`: The MongoDB connection URI of the target instance (e.g., the restored instance for PITR, or the test instance for workload replay).

Example Workflow with Volume Snapshot (PITR):

Let's say you take daily LVM/EBS volume snapshots of your MongoDB data volume at 02:00 AM UTC.  
You need to recover your database to 04:35 AM UTC on June 3rd, 2025\.

1. Restore Snapshot:  
   * Restore the MongoDB data volume from the snapshot taken at `2025-06-03T02:00:00Z` to a new or clean MongoDB instance.  
   * Start this restored MongoDB instance. It will be in the state it was at approximately 02:00 AM.  
2. Apply Oplogs using the CLI tool:  
   * To roll forward from the snapshot time to the desired recovery point:
     ```
     ./pitr_cli myProdCluster mongodb://oplog_store_host/oplogArchiveDB apply \  
         --from_time "2025-06-03T01:59:55Z" \  
         --to_time "2025-06-03T04:35:00Z" \  
         --target_url "mongodb://user:pass@restored_mongo_host:27017/admin"
     ```

     * Note: `--from_time` is set slightly before the snapshot time to ensure all relevant operations around the snapshot point are included.

## **6\. Log Management**

* The script logs its operations to `logs/pitr_cli.log` (located in a `logs` directory one level above the script's directory, e.g., `../logs/pitr_cli.log`).  
* Log rotation is implemented.  
* Log level can be controlled using the `PITR_CLI_LOG_LEVEL` environment variable.

## **7\. Important Considerations**

* `license.key` file: The license file must be present in the same directory as the executable for the tool to function.  
* Data Consistency of Base Backup: For a successful PITR, the base backup or snapshot must be taken from a consistent state of the MongoDB instance. It's crucial to run db.fsyncLock() on the source MongoDB instance before taking the snapshot and `db.fsyncUnlock()` after the snapshot is complete.  
* Oplog Availability: This tool relies on the continuous and complete collection of oplogs by a tool like `MongoBuddy/oplog_collector`. Any gaps in oplog collection will result in an incomplete recovery for that period.

## **8\. Contact & Feedback**

This tool is under active development. For all inquiries, including licensing, feature requests, bug reports, or general feedback, please contact us at:

MongoBuddyNow@gmail.com
