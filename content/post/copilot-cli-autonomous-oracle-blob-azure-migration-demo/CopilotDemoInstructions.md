# Oracle BLOB to Azure Blob Storage — Copilot Demo Instructions

> **Purpose**: These instructions guide GitHub Copilot (or a human developer) through an end-to-end demo that migrates BLOB documents from an Oracle database table into Azure Blob Storage. Every step is designed to be executed sequentially, with Copilot performing the automation while the developer observes and provides input when prompted.

---

## Prerequisites

Before starting, ensure the following tools are available on the machine:

| Tool | Minimum Version | Purpose |
|---|---|---|
| **Podman** | 4.x+ | Run a containerized Oracle database |
| **.NET SDK** | 10.0+ | Build and run the C# console application |
| **Azure CLI (`az`)** | 2.x+ | Provision Azure resources and configure permissions |
| **SQL*Plus or SQLcl** *(optional)* | — | Manual verification of Oracle data (Copilot will use programmatic access) |

The following file must be present in the working directory:

- **`SafeTrackSchema.png`** — An image showing the schema for the `SAFETRACK.SAFETRACK_DOCUMENTS` database table.

---

## Demo Overview

```
┌──────────────┐      ┌──────────────────┐      ┌───────────────────────┐
│  Schema Image│─────▶│  Oracle Database  │─────▶│  Azure Blob Storage   │
│  (.png)      │      │  (Podman container)│      │  (Storage Account)    │
└──────────────┘      └──────────────────┘      └───────────────────────┘
       │                      │                           ▲
       │   Copilot reads      │   10 sample records       │   C# .NET 10 app
       │   and generates DDL  │   with BLOB files          │   reads Oracle,
       │                      │                           │   uploads blobs,
       ▼                      ▼                           │   verifies SHA256
  CREATE TABLE ──────▶ INSERT INTO ───────────────────────┘
```

**What gets built:**

1. A running Oracle database container with a populated `SAFETRACK_DOCUMENTS` table
2. An Azure Storage account with a blob container
3. A .NET 10 console application that migrates BLOBs from Oracle to Azure
4. SHA256-based integrity verification proving files transferred unchanged

---

## Step 1 — Examine the Database Schema Image

**Open and read** the schema image file (`SafeTrackSchema.png`) located in the working directory. This image defines the structure of the **`SAFETRACK.SAFETRACK_DOCUMENTS`** table, including all column names, data types, sizes, and constraints.

Extract the full schema from the image and use it for all subsequent table creation and data generation steps.

**🔵 Checkpoint:** Present the extracted schema to the user in a table format. Briefly summarize what was found (number of columns, primary key, data types). **Ask the user to confirm** the schema looks correct before proceeding.

---

## Step 2 — Pull and Run the Oracle Database Container

Use **Podman** (not Docker) to pull and start a lightweight Oracle database container suitable for development and testing.

**What to do:**

1. Pull the container image `gvenzl/oracle-free:slim`.
2. Run the container with the following configuration:
   - **Container name**: `oracle-safetrack`
   - **Port mapping**: Host port `1521` → Container port `1521`
   - **Root password** (environment variable `ORACLE_PASSWORD`): `SafeTrackDemo123`
   - **Application user** (environment variable `APP_USER`): `safetrack`
   - **Application user password** (environment variable `APP_USER_PASSWORD`): `SafeTrackDemo123`
   - Run in detached mode.
3. Monitor the container logs and **wait** until the message `DATABASE IS READY TO USE!` appears before proceeding. This may take 1–3 minutes.

**🔵 Checkpoint:** Once the database is ready, inform the user that the Oracle container is running and accepting connections. **Ask the user if they are ready** to proceed with creating the table and loading data.

**Connection details for later steps:**

| Property | Value |
|---|---|
| Host | `localhost` |
| Port | `1521` |
| Service Name | `FREEPDB1` |
| User | `safetrack` |
| Password | `SafeTrackDemo123` |
| Connection String | `User Id=safetrack;Password=SafeTrackDemo123;Data Source=//localhost:1521/FREEPDB1;` |

---

## Step 3 — Create the Database Table

Generate and execute a `CREATE TABLE` statement based on the schema from Step 1.

**What to do:**

1. Write a SQL `CREATE TABLE` statement for `SAFETRACK_DOCUMENTS` that matches the schema image exactly, including:
   - All column names, data types, and sizes
   - The primary key constraint `SAFETRACK_DOCUMENTS_PK` on `DOC_SID`
2. Connect to the Oracle container database using the application user credentials from Step 2.
3. Execute the DDL statement.
4. Verify the table was created successfully by describing it or querying `USER_TABLES`.

---

## Step 4 - Generate and Insert 10 Sample Data Records

Populate the table with **10 realistic sample records**, each containing an actual BLOB file in the `DOC_CONTENT` column.

**What to do:**

1. **Generate sample BLOB files** — Create a variety of small binary files to serve as document content. The set should include a realistic mix of file types:
   - **PDF** files (e.g., inspection reports, compliance orders, warnings)
   - **DOCX** files (e.g., safety reports, procedures)
   - **JPG / PNG** files (e.g., site photos, evidence images)
   - **TXT** files (e.g., notes, logs)
   - **XLSX** files (e.g., data exports, checklists)

   Each generated file should be a small but **valid** file of its type (not just random bytes) so that file-type detection and Content-Type mapping work correctly when migrated.

2. **Compose 10 INSERT statements** (or use a bulk-insert approach) that populate every column using the schema extracted from the image in Step 1:
   - `DOC_SID`: Sequential numbers starting from 1001
   - Use a realistic mix of values for all columns based on their data types and purposes as defined in the schema
   - `DOC_CONTENT`: The binary content of one of the generated sample files
   - Include a mix of document type codes (e.g., `IN`, `CO`, `PH`, `WA`, `RP`) distributed across the 10 records
   - Use realistic file names with correct extensions, meaningful descriptions, sample user names, and dates within the last 2 years

3. **Verify** the data by running a count query and a sample SELECT to confirm all 10 records exist and the BLOBs are non-null.

**🔵 Checkpoint:** Present the verification results — total record count, sample records showing SID/type/file name/BLOB size, and the mix of document types. **Ask the user if they are ready** to proceed to Azure setup.

---

## Step 5 — Verify the Azure CLI Is Installed

Before provisioning any Azure resources, confirm that the Azure CLI is available.

**What to do:**

1. Run `az version` to verify the Azure CLI is installed and accessible.
2. If the CLI is **not installed**, inform the user and stop. Provide a link to the [Azure CLI installation page](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
3. If the CLI **is installed**, display the version and proceed.

---

## Step 6 — Confirm Azure Login and Subscription

**What to do:**

1. **Ask the user** to confirm they are logged into the Azure CLI (`az login`) and that the correct Azure subscription is selected.
2. Run `az account show` to display the current subscription name and ID.
3. Present this information to the user and **ask for explicit confirmation** that this is the subscription where resources should be created.
4. If the user needs to switch subscriptions, guide them to use `az account set --subscription "<name or id>"` and re-confirm.

---

## Step 7 — Collect Resource Group and Storage Account Names

**What to do:**

1. **Ask the user** to provide:
   - A **resource group name** (e.g., `rg-safetrack-demo`)
   - A **storage account name** (e.g., `safetrackdemo`) — remind them that storage account names must be 3–24 characters, lowercase letters and numbers only, and globally unique
2. Optionally ask for a **preferred Azure region** (e.g., `eastus`, `westus2`) or suggest a default.
3. Store these values for use in subsequent steps.

---

## Step 8 — Create Azure Resources

Use the Azure CLI to provision the required Azure infrastructure.

**What to do:**

1. **Resource Group** — Check if the resource group already exists. If it does not, create it:
   - Use the name and region provided by the user in Step 7.

2. **Storage Account** — Create a general-purpose v2 storage account:
   - Use the storage account name provided by the user.
   - Place it in the resource group from above.
   - Use Standard_LRS (locally redundant storage) for the demo.
   - Ensure the minimum TLS version is 1.2.
   - **Name conflict handling**: Storage account names must be globally unique across all of Azure. If creation fails due to a name conflict (e.g., `StorageAccountAlreadyTaken`), automatically retry by appending a numeric suffix to the user's chosen name (e.g., `safetrackdemo1`, `safetrackdemo2`, etc.). Keep incrementing and retrying until an available name is found. Inform the user of the final name used.

3. **Blob Container** — Create a blob container within the storage account:
   - Container name: `safetrack-documents`
   - Access level: Private (no anonymous access).

4. **Verify** by listing the container to confirm it was created.

**🔵 Checkpoint:** Summarize all Azure resources created — resource group name and region, storage account name, and blob container name. **Ask the user if they are ready** to proceed with building the .NET migration application.

---

## Step 9 — Create the C# .NET 10 Console Application

Build a .NET 10 console application that reads all records from the Oracle `SAFETRACK_DOCUMENTS` table and uploads each BLOB to Azure Blob Storage.

**What to create:**

### Project Structure

- A new .NET 10 console application solution and project (e.g., `safetrack` solution with an `safetrack` project).

### NuGet Packages

The project should reference the following packages:

| Package | Purpose |
|---|---|
| `Oracle.ManagedDataAccess.Core` | Oracle database connectivity |
| `Azure.Storage.Blobs` | Azure Blob Storage client |
| `Azure.Identity` | `DefaultAzureCredential` for Azure authentication |
| `Microsoft.Extensions.Configuration.Json` | Load settings from `appsettings.json` |

### Configuration File (`appsettings.json`)

The application should be configurable via an `appsettings.json` file with the following settings:

- **Oracle section**: A connection string pointing to the Oracle database.
- **Azure section**:
  - A `StorageConnectionString` field (for connection-string-based authentication).
  - A `StorageAccountUrl` field (for `DefaultAzureCredential`-based authentication, e.g., `https://<account>.blob.core.windows.net`).
  - A `ContainerName` field (default: `safetrack-documents`).

The application should support **either** authentication method:
- If `StorageConnectionString` is provided, use it directly.
- If `StorageAccountUrl` is provided (and `StorageConnectionString` is empty), use `DefaultAzureCredential`.
- If neither is provided, exit with a clear error message.

Populate the `appsettings.json` with the correct values from Steps 2 and 8.

### Application Behavior (`Program.cs`)

The console application should perform the following:

1. **Load configuration** from `appsettings.json`.
2. **Connect to Azure Blob Storage** using the configured authentication method.
3. **Create the blob container** if it does not already exist (private access).
4. **Connect to the Oracle database** using the configured connection string.
5. **Query all records** from `SAFETRACK_DOCUMENTS` where `DOC_CONTENT IS NOT NULL`, ordered by `DOC_SID`.
6. **For each record**:
   - Read all column values.
   - Read the BLOB content from `DOC_CONTENT`.
   - Compute a **SHA256 hash** of the BLOB for integrity verification.
   - Determine the upload blob path using the pattern: `{DOC_TYPE}/{DOC_SID}/{DOC_FILE_NAME}`.
   - Upload the BLOB to Azure Blob Storage with:
     - An appropriate **Content-Type** header based on the file extension (e.g., `.pdf` → `application/pdf`, `.jpg` → `image/jpeg`, `.docx` → the OOXML MIME type, etc.).
     - **Blob metadata** preserving all original Oracle column values (`doc_sid`, `name_sid`, `doc_type`, `to_print_card`, `doc_desc`, `doc_file_name`, `created_by`, `created_on`, `modified_by`, `modified_on`), plus the SHA256 hash and a migration timestamp.
   - Print a status line for each record showing: SID, type, file name, size, success/failure status, and a truncated hash.
7. **Print a summary** showing total documents migrated, total bytes transferred, any failures, and the target container URI.

### Ensuring the `appsettings.json` Is Copied to Output

Configure the project file so that `appsettings.json` is copied to the output directory on build (use `PreserveNewest`).

**🔵 Checkpoint:** Inform the user that the .NET 10 console application has been created. Briefly summarize what was built — project name, NuGet packages added, authentication method configured, and the key behaviors of Program.cs (Oracle read → SHA256 hash → Azure upload with metadata). **Ask the user if they are ready** to proceed with configuring Azure RBAC permissions.

---

## Step 10 — Configure Azure RBAC Permissions

When using `DefaultAzureCredential`, the identity used at runtime must have permission to write blobs.

**What to do:**

1. **Determine the current identity** — Run `az ad signed-in-user show` to get the Object ID of the currently signed-in Azure CLI user. This is the identity that `DefaultAzureCredential` will resolve to when the app runs locally.

2. **Assign the `Storage Blob Data Contributor` role** — Using the Azure CLI, create a role assignment:
   - **Assignee**: The Object ID from above.
   - **Role**: `Storage Blob Data Contributor`.
   - **Scope**: The storage account created in Step 8 (full resource ID: `/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}`).

3. **Wait for propagation** — Inform the user that Azure RBAC role assignments can take **up to 5 minutes** to propagate. Suggest waiting before running the application.

4. **Verify** — Optionally run a quick test (e.g., `az storage blob list`) using the identity to confirm access is working.

---

## Step 11 — Run the Application and Verify Results

**What to do:**

1. **Build the application** — Run `dotnet build` and confirm it compiles without errors.

2. **Ask the user** — Inform the user that the application is ready to run. Ask if they would like you to execute it now.

3. **If the user confirms, run the application** — Execute `dotnet run` from the project directory.

4. **Report the results** — Display:
   - The number of documents migrated successfully.
   - The number of failures (if any).
   - The total data size transferred.
   - The Azure Blob Storage container URI.

5. **Verify integrity** — Confirm that the uploaded blobs match the source data:
   - List all blobs in the Azure Storage container.
   - For each blob, retrieve its metadata (which includes the `sha256_hash` property set during upload).
   - Download each blob and compute a fresh SHA256 hash.
   - Compare the computed hash against the stored metadata hash.
   - Report the verification results: how many blobs matched, and flag any mismatches.

   This SHA256 comparison provides **chain-of-custody verification** that every file arrived in Azure Blob Storage unchanged from its Oracle source.

6. **Summary** — Present a final summary confirming:
   - ✅ Oracle database was provisioned and populated
   - ✅ Azure resources were created
   - ✅ All documents were migrated
   - ✅ Integrity verification passed (all SHA256 hashes match)

---

## Step 12 — Pause Before Cleanup

After a successful migration and verification, **stop and ask the user** to confirm they are ready to proceed with cleanup. Do not begin any cleanup actions until the user explicitly says they are ready.

This gives the presenter time to explore the Azure portal, inspect blobs, review metadata, or discuss the results with the audience before resources are deleted.

---

## Step 13 — Clean Up

After the user confirms they are ready, clean up all resources created during the demo.

**What to do:**

1. **Delete the Azure Storage Account** — Use the Azure CLI to delete the storage account created in Step 8:
   - Run `az storage account delete --name <storage-account-name> --resource-group <resource-group-name> --yes`.
   - Confirm the deletion completed successfully.

2. **Ask about the Resource Group** — **Ask the user** whether the resource group should also be deleted:
   - If the user confirms, delete it: `az group delete --name <resource-group-name> --yes --no-wait`.
   - If the user declines, leave the resource group in place.

3. **Stop and remove the Oracle container** — Clean up the Podman container:
   - Run `podman stop oracle-safetrack && podman rm oracle-safetrack`.

4. **Delete local demo files** — Remove all files and folders in the working directory that were created during the demo, **except** the following which must be preserved:
   - `CopilotDemoInstructions.md` — this instructions file
   - `DemoScript.md` — the presenter demo script
   - `SafeTrackSchema.png` — the database schema image

   This includes deleting:
   - The .NET solution file (e.g., `safetrack.sln` or `safetrack.slnx` if it was created/modified)
   - The .NET project folder (e.g., `safetrack/`)
   - Any temporary utility projects or scripts created during setup (e.g., `setup_db/`)
   - Any other generated files or folders

5. **Verify** — List the contents of the working directory to confirm only `CopilotDemoInstructions.md`, `DemoScript.md`, and `SafeTrackSchema.png` remain.

---

## Quick Reference

| Item | Value |
|---|---|
| Oracle Container Image | `gvenzl/oracle-free:slim` |
| Container Runtime | Podman |
| Oracle Port | `1521` |
| Oracle Service | `FREEPDB1` |
| Oracle User | `safetrack` |
| Oracle Password | `SafeTrackDemo123` |
| .NET Version | 10.0 |
| Blob Container Name | `safetrack-documents` |
| Blob Path Pattern | `{DOC_TYPE}/{DOC_SID}/{DOC_FILE_NAME}` |
| Auth Method | `DefaultAzureCredential` (preferred) or connection string |
| Required RBAC Role | `Storage Blob Data Contributor` |
| Integrity Check | SHA256 hash comparison |
