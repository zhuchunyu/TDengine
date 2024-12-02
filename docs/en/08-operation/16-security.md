---
title: Advanced Security Options
slug: /operations-and-maintenance/advanced-security-options
---

In addition to traditional user and permission management, TDengine offers other security policies, such as IP whitelisting, audit logs, and data encryption, which are unique features of TDengine Enterprise. The whitelisting feature was first released in version 3.2.0.0, the audit logs in version 3.1.1.0, and database encryption in version 3.3.0.0. It is recommended to use the latest version.

## IP Whitelisting

IP whitelisting is a network security technology that allows IT administrators to control "who" can access systems and resources, enhancing the security of database access and preventing external malicious attacks. IP whitelisting creates a list of trusted IP addresses, assigning them as unique identifiers for users, and only allows these IP addresses to access the target server. It is important to note that user permissions are independent of the IP whitelist and are managed separately. Below are the specific methods for configuring IP whitelisting.

The SQL to add an IP whitelist is as follows.

```sql
create user test pass password [sysinfo value] [host host_name1[,host_name2]] 
alter user test add host host_name1
```

The SQL to query the IP whitelist is as follows.

```sql
SELECT TEST, ALLOWED_HOST FROM INS_USERS;
SHOW USERS;
```

The command to delete an IP whitelist is as follows.

```sql
ALTER USER TEST DROP HOST HOST_NAME1
```

## Audit Logs

TDengine records and manages user operations, then sends these as audit logs to taosKeeper, which saves them to any TDengine cluster. Administrators can use audit logs for security monitoring and historical tracing. The audit log feature in TDengine can be easily turned on or off by modifying the TDengine configuration file and restarting the service. The configuration for audit logs is described below.

### taosd Configuration

Audit logs are generated by the database service taosd, and the corresponding parameters must be configured in the `taos.cfg` configuration file, as shown in the table below.

|  Parameter Name   |                      Parameter Meaning                       |
| :---------------: | :----------------------------------------------------------: |
|       audit       | Whether to enable audit logs, with a default value of 0. 1 to enable, 0 to disable |
|    monitorFqdn    | FQDN of the server where the taosKeeper receiving the audit logs is located |
|    monitorPort    | Port used by the taosKeeper service receiving the audit logs |
| monitorCompaction |          Whether to compress data during reporting           |

### taosKeeper Configuration

In the `keeper.toml` configuration file of taosKeeper, configure parameters related to audit logs, as shown in the table below.

| Parameter Name |                      Parameter Meaning                       |
| :------------: | :----------------------------------------------------------: |
|    auditDB     | The name of the database for storing audit logs, with a default value of "audit". After receiving the reported audit logs, taosKeeper will check if this database exists; if it does not, it will create it automatically. |

### Data Format

The format of the reported audit logs is as follows:

```json
{
    "ts": timestamp,
    "cluster_id": string,
    "user": string,
    "operation": string,
    "db": string,
    "resource": string,
    "client_add": string,
    "details": string
}
```

### Table Structure

taosKeeper will automatically create a supertable in the corresponding database based on the reported audit data to store the data. The definition of this supertable is as follows:

```sql
CREATE STABLE operations(ts timestamp, details VARCHAR(64000), User VARCHAR(25), Operation VARCHAR(20), db VARCHAR(65), resource VARCHAR(193), client_add(25)) TAGS (clusterID VARCHAR(64));
```

Where:

1. `db` refers to the database involved in the operation, and `resource` refers to the resource involved in the operation.
2. `User` and `Operation` are data columns that indicate which user performed what operation on the object.
3. `timestamp` is the timestamp column indicating when the operation occurred.
4. `details` contains supplementary details about the operation, which is the SQL statement executed in most operations.
5. `client_add` refers to the client address, including IP and port.

### Operation List

The current list of operations recorded in the audit logs and the meaning of each field in each operation is shown in the table below (Note: the meaning of the `user` field, timestamp field, and `client_add` is the same in all operations, so they are not included in the table).

| Operation             | Operation          | DB            | Resource             | Details                                                      |
| --------------------- | ------------------ | ------------- | -------------------- | ------------------------------------------------------------ |
| create database       | createDB           | db name       | NULL                 | SQL                                                          |
| alter database        | alterDB            | db name       | NULL                 | SQL                                                          |
| drop database         | dropDB             | db name       | NULL                 | SQL                                                          |
| create stable         | createStb          | db name       | stable name          | SQL                                                          |
| alter stable          | alterStb           | db name       | stable name          | SQL                                                          |
| drop stable           | dropStb            | db name       | stable name          | SQL                                                          |
| create user           | createUser         | NULL          | User created         | User property parameters (excluding password)                |
| alter user            | alterUser          | NULL          | User modified        | Records modified parameters and new values for password changes (excluding password); other operations record SQL |
| drop user             | dropUser           | NULL          | User deleted         | SQL                                                          |
| create topic          | createTopic        | topic DB      | Created topic name   | SQL                                                          |
| drop topic            | dropTopic          | topic DB      | Deleted topic name   | SQL                                                          |
| create dnode          | createDnode        | NULL          | IP:Port or FQDN:Port | SQL                                                          |
| drop dnode            | dropDnode          | NULL          | dnodeId              | SQL                                                          |
| alter dnode           | alterDnode         | NULL          | dnodeId              | SQL                                                          |
| create mnode          | createMnode        | NULL          | dnodeId              | SQL                                                          |
| drop mnode            | dropMnode          | NULL          | dnodeId              | SQL                                                          |
| create qnode          | createQnode        | NULL          | dnodeId              | SQL                                                          |
| drop qnode            | dropQnode          | NULL          | dnodeId              | SQL                                                          |
| login                 | login              | NULL          | NULL                 | appName                                                      |
| create stream         | createStream       | NULL          | Created stream name  | SQL                                                          |
| drop stream           | dropStream         | NULL          | Deleted stream name  | SQL                                                          |
| grant privileges      | grantPrivileges    | NULL          | User granted         | SQL                                                          |
| remove privileges     | revokePrivileges   | NULL          | User revoked         | SQL                                                          |
| compact database      | compact            | database name | NULL                 | SQL                                                          |
| balance vgroup leader | balanceVgroupLead  | NULL          | NULL                 | SQL                                                          |
| restore dnode         | restoreDnode       | NULL          | dnodeId              | SQL                                                          |
| redistribute vgroup   | redistributeVgroup | NULL          | vgroupId             | SQL                                                          |
| balance vgroup        | balanceVgroup      | NULL          | vgroupId             | SQL                                                          |
| create table          | createTable        | db name       | NULL                 | table name                                                   |
| drop table            | dropTable          | db name       | NULL                 | table name                                                   |

### Viewing Audit Logs

After both `taosd` and `taosKeeper` are correctly configured and started, various operations in the system (as shown in the table above) will be recorded and reported in real-time as the system runs. Users can log into `taosExplorer`, click on "System Management" → "Audit" page to view the audit logs; they can also directly query the corresponding databases and tables in the TDengine CLI.

## Data Encryption

TDengine supports Transparent Data Encryption (TDE), which encrypts static data files to prevent potential attackers from bypassing the database and directly reading sensitive information from the file system. The database access program is completely unaware of this; applications do not need any modifications or recompilation to work directly with the encrypted database, supporting encryption algorithms such as the national standard SM4. In transparent encryption, database key management and the scope of database encryption are two of the most important topics. TDengine encrypts the database key using machine codes, storing it locally rather than in a third-party manager. When data files are copied to other machines, the change in machine codes means the database key cannot be obtained, and thus the data files cannot be accessed. TDengine encrypts all data files, including write-ahead log files, metadata files, and time-series data files. After encryption, the data compression rate remains unchanged, and there is only a slight decrease in write and query performance.

### Configuring Keys

Key configuration can be done in two ways: offline settings and online settings.

**Method One: Offline Setting**. You can configure keys separately for each node through offline settings, with the command as follows.

```shell
taosd -y {encryptKey}
```

**Method Two: Online Setting**. When all nodes in the cluster are online, you can use SQL to configure the keys, with the SQL as follows.

```sql
create encrypt_key {encryptKey};
```

The online setting method requires that all nodes that have joined the cluster have not used the offline setting method to generate keys; otherwise, the online setting will fail. Successfully setting the online key will also automatically load and use the key.

### Creating Encrypted Databases

TDengine supports creating encrypted databases via SQL, with the SQL as follows.

```sql
create database [if not exists] db_name [database_options]
database_options:
 database_option ...
database_option: {
 encrypt_algorithm {'none' |'sm4'}
}
```

The main parameters are described as follows.

- `encrypt_algorithm`: Specifies the encryption algorithm used for data. The default is `none`, meaning no encryption. `sm4` indicates the use of the SM4 encryption algorithm.

### Viewing Encryption Configuration

Users can query the current encryption configuration of databases by querying the system database `ins_databases`, with the SQL as follows.

```sql
select name, `encrypt_algorithm` from ins_databases;
              name              | encrypt_algorithm |
=====================================================
 power1                         | none              |
 power                          | sm4               |
```

### Viewing Node Key Status

You can check the node key status with the following SQL command:

```sql
show encryptions;

select * from information_schema.ins_encryptions;
  dnode_id   |           key_status           |
===============================================
           1 | loaded                         |
           2 | unset                          |
           3 | unknown                        |
```

The `key_status` has three possible values:

- When a node has not set a key, the status column displays `unset`.
- When the key is successfully validated and loaded, the status column displays `loaded`.
- When the node is not started, and the key's status cannot be determined, the status column displays `unknown`.

### Updating Key Configuration

When the hardware configuration of a node changes, you need to update the key using the following command, which is the same as the offline configuration command:

```shell
taosd -y  {encryptKey}
```

To update the key configuration, you must first stop `taosd`, and use exactly the same key; that is, the key cannot be modified after the database is created.
