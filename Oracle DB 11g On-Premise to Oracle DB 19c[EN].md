# Migration of On-Premise Oracle Database 11g Release 11.2.0.4.0 to Oracle Database 19c Standard Edition 2 Version 19.28.0.0.0

**Prepared by:** Gustavo H Meireles | [LinkedIn Profile](https://www.linkedin.com/in/gustavo-henrique-e-m-meireles/)

---

## Table of Contents

- [Introduction](#introduction)
- [SECTION I: Environment Assessment & Target Provisioning](#section-i-environment-assessment--target-provisioning)
  - [1. General Assessment of Hosts Involved in the Operation](#1-general-assessment-of-hosts-involved-in-the-operation)
    - [SOURCE Environment (ORIGEM)](#source-environment-origem)
    - [TARGET Environment (DESTINO)](#target-environment-destino)
  - [2. Parametrization and Configuration of the PRODUCTION Environment on TARGET](#2-parametrization-and-configuration-of-the-production-environment-on-target)
  - [3. Creation of the STAGING Environment (HOMOLOGACAO) on TARGET](#3-creation-of-the-staging-environment-homologacao-on-target)
- [SECTION II: Data Migration Execution & Validation](#section-ii-data-migration-execution--validation)
  - [1. Data Export from the SOURCE Environment](#1-data-export-from-the-source-environment)
  - [2. Data Import and Validation in the Production Environment on TARGET](#2-data-import-and-validation-in-the-production-environment-on-target)
  - [3. Data Import and Validation in the STAGING Environment on TARGET](#3-data-import-and-validation-in-the-staging-environment-on-target)
- [SECTION III: Conclusion and Recommendations](#section-iii-conclusion-and-recommendations)
  - [Conclusion](#conclusion)
  - [Recommendations](#recommendations)

---

## Introduction

This technical documentation details the end-to-end migration process of a single-instance **Oracle Database 11g Release 11.2.0.4.0 On-Premise** environment (referred to throughout this document as **SOURCE** / *ORIGEM*) to an **Oracle Database 19c Standard Edition 2 Version 19.28.0.0.0 Multitenant (CDB/PDB)** environment deployed in a Private Cloud (referred to as **TARGET** / *DESTINO*).

### Key Technologies Employed
- **OpenVPN**: Secure encrypted tunnel connectivity between SOURCE and TARGET hosts.
- **Oracle Data Pump (`EXPDP` / `IMPDP`)**: Logical backup, export, and import of database objects and schemas.
- **Oracle Cloud Infrastructure (OCI) Object Storage**: High-throughput file transfer medium for compressed migration dumps.
- **Red Hat Ansible Automation Platform**: Infrastructure-as-Code (IaC) automation for target host provisioning, operating system dependencies, and Oracle 19c CDB/PDB software setup prior to data import.

### Migration Strategy & Rationale
The adoption of logical export and import via Data Pump (`expdp`/`impdp`) was selected over physical migration techniques (such as RMAN restore or transportable tablespaces) due to architectural shifts between the two environments:
1. **Architectural Transition**: Moving from a traditional non-CDB single instance (11g) to a Multitenant Container Database architecture (19c CDB/PDB).
2. **Selective Schema Deployment**: Data Pump allowed exporting multiple application schemas (`EMPRESA01`, `EMPRESA02`, `EMPRESA03`, `BI01`, `BI02`, `FV01`, `FV02`, `HOMOLOGACAO`) from SOURCE and selectively routing them into distinct target environments on TARGET without requiring a full database physical clone.
3. **Segregation of Staging**: The staging/homologation schema (`HOMOLOGACAO`) was isolated into a dedicated Pluggable Database (`PDB TESTE`), while production schemas were loaded into the primary production PDB (`ORCL`).

### Benefits of Pluggable Database (PDB) Segregation
Deploying the staging schema into a dedicated PDB (`PDB TESTE`) within Oracle 19c Multitenant offers substantial operational advantages:
- **Logical Isolation**: Prevents test queries, schema modifications, or experimental workloads from interfering with production operations.
- **Enhanced Security & Governance**: Enables independent access policies, distinct database links, and tailored audit trails for production versus staging.
- **Operational Independence**: Facilitates isolated backup and restore schedules for testing without impacting production availability.
- **Resource Management**: Prevents non-production workloads from consuming excessive CPU, memory, or I/O resources assigned to production.
- **Risk Mitigation**: Provides an isolated environment for validating application upgrades, patch testing, and performance tuning prior to production release.

### Data Protection & Compliance Notice
In accordance with legal mandates under the Brazilian General Data Protection Law (LGPD - *Lei Geral de Proteção de Dados*, Law No. 13.709/2018) and strict corporate security policies, sensitive credentials, passwords, machine IDs, and proprietary strings have been redacted and masked as `xxxxxxxxxxxxxxxx`.

---

## SECTION I: Environment Assessment & Target Provisioning

### 1. General Assessment of Hosts Involved in the Operation

#### SOURCE Environment (ORIGEM)

##### Host Hardware & Operating System
- **Manufacturer / Product**: Dell Inc. PowerEdge T140
- **Static Hostname**: ORIGEM
- **Chassis / Machine Type**: Server
- **Operating System**: Oracle Linux Server 7.6 (x86_64)
- **Kernel Version**: Linux 4.14.35-1818.3.3.el7uek.x86_64
- **Processor**: Intel(R) Xeon(R) E-2124 CPU @ 3.30GHz
- **CPU Configuration**: 1 Socket | 4 Cores per Socket | 1 Thread per Core (Total 4 vCPUs)
- **Total Memory**: 39,941 MB
- **Swap Space**: 16,383 MB

##### Database Configuration & Properties
Database parameters retrieved via `SELECT property_name, property_value FROM database_properties;`:

| Property Name | Property Value |
| :--- | :--- |
| `NLS_RDBMS_VERSION` | `11.2.0.4.0` |
| `GLOBAL_DB_NAME` | `ORCL` |
| `DEFAULT_TBS_TYPE` | `SMALLFILE` |
| `DEFAULT_PERMANENT_TABLESPACE` | `USERS` |
| `DEFAULT_TEMP_TABLESPACE` | `TEMP` |
| `NLS_CHARACTERSET` | `WE8MSWIN1252` |
| `NLS_NCHAR_CHARACTERSET` | `AL16UTF16` |
| `NLS_LANGUAGE` | `AMERICAN` |
| `NLS_TERRITORY` | `AMERICA` |
| `NLS_CURRENCY` | `$` |
| `NLS_NUMERIC_CHARACTERS` | `.,` |
| `NLS_DATE_FORMAT` | `DD-MON-RR` |
| `NLS_DATE_LANGUAGE` | `AMERICAN` |
| `NLS_SORT` | `BINARY` |
| `NLS_LENGTH_SEMANTICS` | `BYTE` |
| `DBTIMEZONE` | `-03:00` |
| `EXPORT_VIEWS_VERSION` | `8` |
| `DICT.BASE` | `2` |

#### TARGET Environment (DESTINO)

##### Host Hardware & Operating System
- **Manufacturer / Product**: VMware, Inc. VMware7,1
- **Static Hostname**: DESTINO
- **Virtualization Type**: VMware VM
- **Operating System**: Oracle Linux Server 8.10 (x86_64)
- **Kernel Version**: Linux 5.15.0-322.203.3.2.el8uek.x86_64
- **Processor**: AMD EPYC 9354 32-Core Processor
- **CPU Configuration**: 1 Socket | 1 Core per Socket | 3 Threads per Core (Total 3 vCPUs)
- **Total Memory**: 14,944 MB
- **Swap Space**: 8,191 MB

##### Database Configuration & Properties
Database parameters retrieved via `SELECT property_name, property_value FROM database_properties;`:

| Property Name | Property Value |
| :--- | :--- |
| `NLS_RDBMS_VERSION` | `19.0.0.0.0` |
| `GLOBAL_DB_NAME` | `ORCL` |
| `LOCAL_UNDO_ENABLED` | `TRUE` |
| `MAX_PDB_SNAPSHOTS` | `8` |
| `MAX_STRING_SIZE` | `STANDARD` |
| `DEFAULT_TBS_TYPE` | `SMALLFILE` |
| `DEFAULT_PERMANENT_TABLESPACE` | `USERS` |
| `DEFAULT_TEMP_TABLESPACE` | `TEMP` |
| `NLS_CHARACTERSET` | `WE8MSWIN1252` |
| `NLS_NCHAR_CHARACTERSET` | `AL16UTF16` |
| `NLS_LANGUAGE` | `BRAZILIAN PORTUGUESE` |
| `NLS_TERRITORY` | `BRAZIL` |
| `NLS_CURRENCY` | `R$` |
| `NLS_NUMERIC_CHARACTERS` | `,.` |
| `NLS_DATE_FORMAT` | `DD/MM/RR` |
| `NLS_DATE_LANGUAGE` | `BRAZILIAN PORTUGUESE` |
| `NLS_SORT` | `WEST_EUROPEAN` |
| `NLS_LENGTH_SEMANTICS` | `BYTE` |
| `DBTIMEZONE` | `-03:00` |
| `DST_PRIMARY_TT_VERSION` | `44` |

---

### 2. Parametrization and Configuration of the PRODUCTION Environment on TARGET

The target host provisioning, operating system software dependencies, and Oracle 19c database instance creation were automated using **Red Hat Ansible Automation Platform**.

1. **OS Installation & Connectivity**: Oracle Linux 8.10 base deployment was completed according to standard OS installation specifications. OpenVPN connectivity was established to enable secure remote orchestration.
2. **Ansible Playbook Execution**: The automation workflow executed the setup of Oracle Home, creation of the Container Database (`ORACDB`), configuration of Pluggable Database (`ORCL`), tablespace allocation, and initialization parameter configuration.

#### Ansible Execution JSON Configuration Payload
The following JSON payload was passed into the Ansible automation workflow with `limit` targeted strictly at the **TARGET** host:

```json
{
  "root_passwd": "xxxxxxxxxxxxxx",
  "oracle_passwd": "xxxxxxxxxxxxxx",
  "software_in_stage": true,
  "particao_princ_bd": "u01",
  "dir_stage": "u01",
  "install_through_vpn": true,
  "device_persistence": "nothing",
  "autostartup_service": true,
  "role_separation": false,
  "configure_motd": true,
  "create_pdb": true,
  "configure_samba": false,
  "samba_name": "ORCL",
  "samba_dir": "/sistema/ORCL",
  "samba_group": "ORCL",
  "samba_user": "admorcl",
  "samba_passwd": "xxxxxxxxxxxxxx",
  "exec_datapatch": true,
  "oracle_home_db": "/u01/app/oracle/19.3.0.0/db_1",
  "oracle_sid": "ORACDB",
  "oracle_version": "19.0.0.0",
  "oracle_databases": [
    {
      "home": "db_1",
      "oracle_version_db": "19.3.0.0",
      "oracle_edition": "SE2",
      "cdb_name": "ORACDB",
      "oracle_db_passwd": "xxxxxxxxxxxxxx",
      "oracle_db_type": "SI",
      "container_db": true,
      "pdb_name": "ORCL",
      "oracle_pdb_passwd": "xxxxxxxxxxxxxx",
      "storage_type": "FS",
      "service_name": "ORCL",
      "oracle_db_mem_totalmb": 12000,
      "oracle_database_type": "MULTIPURPOSE",
      "redolog_size_in_mb": 200,
      "listener_name": "LISTENER",
      "state": "present",
      "characterset": "WE8MSWIN1252",
      "ncharacterset": "AL16UTF16",
      "automatic_memory_management": true,
      "oracle_init_params": "processes=2000,open_cursors=5000,session_cached_cursors=400,sec_case_sensitive_logon=FALSE, nls_territory=AMERICA, nls_language=AMERICAN",
      "dbca_templatename": "dbca-create-cdb.rsp.19.3.0.0.dbt",
      "db_recovery_size_mb": 204800,
      "ts_indice_size_mb": 5000,
      "ts_dados_size_mb": 5000,
      "export_dir": "/u02/oradata/ORACDB/ORCL/export",
      "exporta_db_passwd": "xxxxxxxxxxxxxx",
      "system_db_passwd": "xxxxxxxxxxxxxx",
      "schema": " xxxxxxxxxxxxxx ",
      "schema_db_passwd": "xxxxxxxxxxxxxx"
    }
  ],
  "oracle_dbf_dir_asm": "+DATA",
  "oracle_reco_dir_asm": "+FRA",
  "configure_backup": true,
  "enable_archivelog": false,
  "db_for_ORCLhor": true,
  "db_for_debx": false
}
```

---

### 3. Creation of the STAGING Environment (HOMOLOGACAO) on TARGET

To ensure complete isolation between production and non-production workloads, a dedicated pluggable database named `TESTE` was provisioned from `PDB$SEED`.

```sql
-- Step 1: Create Pluggable Database TESTE from PDB$SEED
CREATE PLUGGABLE DATABASE TESTE ADMIN USER TESTE IDENTIFIED BY HOMOLOGA
FILE_NAME_CONVERT = ('/u01/oradata/ORACDB/pdbseed/', '/u01/oradata/ORACDB/TESTE/');

-- Step 2: Provision Data Tablespace for Staging
CREATE TABLESPACE TS_DADOS DATAFILE '/u01/oradata/ORACDB/TESTE/ts_dados01.dbf'
SIZE 1G AUTOEXTEND ON NEXT 1G MAXSIZE UNLIMITED;

-- Step 3: Provision Index Tablespace for Staging
CREATE TABLESPACE TS_INDICE DATAFILE '/u01/oradata/ORACDB/TESTE/ts_indice01.dbf'
SIZE 1G AUTOEXTEND ON NEXT 1G MAXSIZE UNLIMITED;

-- Step 4: Create Staging Schema User and Assign Default Storage
CREATE USER HOMOLOGA IDENTIFIED BY HOMOLOGA DEFAULT TABLESPACE TS_DADOS TEMPORARY TABLESPACE TEMP;
```

---

## SECTION II: Data Migration Execution & Validation

### 1. Data Export from the SOURCE Environment

The database export was performed directly on the **SOURCE** host using Oracle Data Pump Export (`expdp`).

#### Step 1: Full Logical Schema Export (`expdp`)
Schemas included in the export dump: `EMPRESA01`, `EMPRESA02`, `EMPRESA03`, `BI01`, `BI02`, `FV01`, `FV02`, and `HOMOLOGACAO`.

```bash
$ expdp EXPORTA/xxxxxxxxxxxxxx \
  dumpfile=EXPDP_ORCL_FULL_07072026-22h.dmp \
  logfile=EXPDP_ORCL_07072026.log \
  directory=export_dir \
  exclude=statistics \
  SCHEMAS=EMPRESA01, EMPRESA02, EMPRESA03, BI01, BI02, FV01, FV02, HOMOLOGACAO
```
- **Estimated Execution Time**: `03h:23m:57s`

#### Step 2: Dump File Compression (`tar`)
To optimize transmission speed over cloud object storage, the dump and log files were compressed into a gzip archive:

```bash
$ tar -zcvf EXPDP_ORCL_FULL_07072026-22h.tar.gz \
  EXPDP_ORCL_FULL_07072026-22h.log \
  EXPDP_ORCL_FULL_07072026-22h.dmp
```
- **Estimated Execution Time**: `29m:57s`

#### Step 3: Upload to Oracle Cloud Infrastructure (OCI) Object Storage (`rclone`)
The compressed archive was uploaded to Oracle OCI Object Storage via `rclone` to serve as a high-bandwidth transfer stage:

```bash
$ rclone copy EXPDP_ORCL_FULL_07072026-22h.tar.gz ocistorage:xxxxxxxxxxxxxx \
  --no-check-certificate --progress
```
- **Estimated Execution Time**: `06m:50s`

---

### 2. Data Import and Validation in the Production Environment on TARGET

#### Step 1: Download Dump File from OCI Object Storage
On the **TARGET** host, the file was fetched from OCI Object Storage:

```bash
$ rclone copy ocistorage:xxxxxxxxxxxxxx/EXPDP_ORCL_FULL_07072026-22h.tar.gz ./ --progress
```
- **Estimated Execution Time**: `10m:22.7s`

#### Step 2: Decompress Archive
```bash
$ tar -xvf EXPDP_ORCL_FULL_07072026-22h.tar.gz
```
- **Estimated Execution Time**: `50m:00s`

#### Step 3: Production Schema Data Import (`impdp`)
Import production schemas (`EMPRESA01`, `EMPRESA02`, `EMPRESA03`, `BI01`, `BI02`, `FV01`, `FV02`) into the production PDB (`ORCL`):

```bash
$ impdp EXPORTA/xxxxxxxxxxxxxx@ORCL \
  dumpfile=EXPDP_ORCL_FULL_07072026-22h.dmp \
  logfile=IMPDP_ORCL_08072026.log \
  directory=EXPORT_DIR \
  TRANSFORM=OID:N \
  SCHEMAS=EMPRESA01, EMPRESA02, EMPRESA03, BI01, BI02, FV01, FV02
```
- **Estimated Execution Time**: `01h:11m:53s`

#### Step 4: Recompile Invalid Database Objects
Following import, invalid PL/SQL packages, views, and triggers were recompiled using Oracle's standard utility script:

```sql
@?/rdbms/admin/utlrp.sql
```

#### Step 5: Gather Database & Optimizer Statistics
Fresh optimizer statistics were gathered across the database, data dictionary, and fixed objects to ensure optimal query execution plans:

```sql
-- Gather full database statistics
EXEC DBMS_STATS.GATHER_DATABASE_STATS;
-- Execution Time: 00:21:41.64

-- Gather data dictionary statistics
EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
-- Execution Time: 00:00:03.90

-- Gather fixed objects statistics
EXEC DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;
-- Execution Time: 00:00:40.20
```

---

### 3. Data Import and Validation in the STAGING Environment on TARGET

The staging schema (`HOMOLOGACAO`) was imported exclusively into the dedicated pluggable database `TESTE` (`@TESTE`).

#### Step 1: Staging Schema Import (`impdp`)
```bash
$ impdp EXPORTA/xxxxxxxxxxxxxx@TESTE \
  dumpfile=EXPDP_ORCL_FULL_07072026-22h.dmp \
  logfile=IMPDP_TESTE_08072026.log \
  directory=EXPORT_DIR \
  TRANSFORM=OID:N \
  SCHEMAS=HOMOLOGACAO
```
- **Estimated Execution Time**: `31m:15s`

#### Step 2: Recompile Invalid Objects in Staging
```sql
@?/rdbms/admin/utlrp.sql
```

---

## SECTION III: Conclusion and Recommendations

### Conclusion

The database migration from **Oracle Database 11g Release 11.2.0.4.0** to **Oracle Database 19c Standard Edition 2 (19.28.0.0.0)** was successfully completed, meeting all predefined technical, performance, and operational requirements.

Key project milestones and outcomes include:
1. **Successful Architectural Modernization**: Seamless transition from a legacy non-CDB single instance to a modern Oracle Multitenant CDB/PDB architecture.
2. **Infrastructure Automation**: Utilizing Red Hat Ansible Automation Platform ensured a standardized, repeatable, and error-free environment build on Oracle Linux 8.10.
3. **Secure & High-Performance Data Transport**: Leveraging OpenVPN encrypted tunnels and Oracle Cloud Infrastructure (OCI) Object Storage guaranteed fast and secure data transport between on-premise hardware and private cloud infrastructure.
4. **Logical Workload Segregation**: Using Data Pump schema filtering enabled clear physical and logical separation between production schemas (`ORCL` PDB) and non-production testing workloads (`TESTE` PDB), adhering to Oracle Multitenant best practices.
5. **Data Integrity & Performance Readiness**: Object recompilation via `utlrp.sql` and comprehensive optimizer statistics collection (`DBMS_STATS`) verified database health, structural consistency, and execution plan stability post-migration.

---

### Recommendations

To maintain optimal operational health, security, and performance on the new Oracle 19c platform, the following administrative and governance practices are recommended:

1. **Continuous Backup & Recovery Governance**:
   - Establish automated, periodic Oracle RMAN backup policies for both Container (`CDB$ROOT`) and Pluggable Databases (`ORCL` and `TESTE`).
   - Conduct periodic restore and point-in-time recovery validations in the staging environment.
2. **Environment Isolation Protocol**:
   - Maintain strict operational separation between production (`ORCL`) and staging (`TESTE`).
   - All application patches, database updates, and schema alterations must be validated in `PDB TESTE` prior to production deployment.
3. **Database Maintenance & Patch Management**:
   - Apply quarterly Oracle Release Updates (RUs) and security patches to keep the database environment secure and compliant.
   - Routinely run statistics gathering procedures (`DBMS_STATS`) following bulk data modifications.
4. **Security & Access Control**:
   - Implement fine-grained access control (FGAC), audit policies, and strict password management in compliance with LGPD requirements.
   - Periodically review database user privileges and grant assignments.
5. **Resource & Capacity Monitoring**:
   - Monitor storage growth, tablespace usage, and CPU/memory allocation on the target host to ensure adequate headroom as transaction volumes scale.
   - Utilize automated monitoring tools for proactive alert management on database events.
