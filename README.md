# mac-ccba-n8n-edw

An [n8n](https://n8n.io/) workflow that extracts CCBA (Coca-Cola Beverages Africa) adherence data from a MySQL data warehouse and uploads the result as a CSV file to Google Drive on a schedule.

## Overview

The workflow (`edw-workflow.json`) runs on a schedule, defines one or more extraction jobs, executes the matching stored procedure against MySQL, normalizes the result set into individual rows, converts it to a CSV file, and uploads it to a Google Drive folder. Failures at each stage trigger an email notification via SMTP.

## Workflow flow

```
Scheduled Trigger
    └─> Define Jobs (sets stored procedure + output file prefix)
          └─> CALL Proc (executes the stored procedure in MySQL)
                ├─> Normalize – Flatten rows into items
                │     ├─> Convert to File (CSV)
                │     │     └─> Upload file (Google Drive)
                │     └─> [on error] Format/Normalisation node failed (email)
                └─> [on error] Call Proc node failed (email)
```

Additional nodes present but currently **disabled**:
- **FTP Upload CSV** — uploads the generated CSV to an FTP server instead of / in addition to Google Drive.
- **SMTP Success** — sends a success email once the FTP upload completes.
- **FTP Server node error** — sends an email if the FTP server is unreachable.

## Nodes

| Node | Type | Purpose |
|---|---|---|
| Scheduled Trigger | `scheduleTrigger` | Kicks off the workflow on a configured interval |
| Define Jobs | `code` | Defines the job(s) to run — stored procedure call and output file name prefix |
| CALL Proc | `mySql` | Executes the stored procedure for the job (e.g. `sp_adherence_ff`) |
| Normalize – Flatten rows into items | `code` | Flattens the stored procedure result set into individual n8n items |
| Convert to File | `convertToFile` | Converts the normalized items into a CSV file |
| Upload file | `googleDrive` | Uploads the CSV to a Google Drive folder |
| Call Proc node failed | `emailSend` | Notifies on stored procedure failure |
| Format/Normalisation node failed | `emailSend` | Notifies on data normalization failure |
| FTP Upload CSV *(disabled)* | `ftp` | Alternate delivery to an FTP server |
| SMTP Success *(disabled)* | `emailSend` | Notifies on successful FTP upload |
| FTP Server node error *(disabled)* | `emailSend` | Notifies if the FTP server is unreachable |

## Requirements

- An n8n instance (self-hosted or cloud)
- Credentials configured in n8n for:
  - **MySQL** — read access to the CCBA reporting database and its stored procedures
  - **Google Drive OAuth2** — access to the target Drive and destination folder
  - **SMTP** — for failure/success email notifications (optional, only needed if email nodes are enabled)

## Setup

1. In n8n, go to **Workflows → Import from File** and select `edw-workflow.json`.
2. Reassign the credentials on each node (MySQL, Google Drive, SMTP) to your own n8n credential entries — the IDs in this export point to credentials in the original instance and will not resolve elsewhere.
3. Update the **Scheduled Trigger** interval to the desired run frequency.
4. Review **Define Jobs** and adjust the stored procedure call(s) and output file prefix as needed.
5. Confirm the **Upload file** node's target Drive/folder.
6. Enable/disable the FTP and email notification nodes as required for your environment.
7. Activate the workflow.

## Notes

- Email addresses and the SMTP2GO credential in this export are specific to the original MacMobile n8n instance and should be updated before use elsewhere.
- The `CALL Proc` and `Normalize` nodes are configured with `continueErrorOutput`, so failures route to their respective error/notification branches instead of stopping the workflow.
