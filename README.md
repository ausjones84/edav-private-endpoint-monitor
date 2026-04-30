# EDAV Private Endpoint Monitor

> **Azure cost-saving automation** — Identifies disconnected private endpoints, validates
> backend resources and Terraform ownership, generates colour-coded Excel reports, and
> optionally emails a summary. **Nothing is deleted without explicit approval.**

---

## What This Tool Does

Your EDAV Resource Monitor (Critical > Network findings) is showing a large number of
**disconnected private endpoints** that are costing money every month. This tool:

1. Reads your CSV/Excel list of endpoints from EDAV
2. Logs into Azure using your SU account (no credentials stored)
3. Scans each endpoint across one or more subscriptions
4. Checks whether the **backend resource** (Key Vault, Storage, SQL, etc.) still exists
5. Checks whether the endpoint is **managed by Terraform** (so you dont accidentally delete it)
6. Produces a professional **colour-coded Excel report** with three tabs:
   - Summary (counts by action)
   - All Endpoints (full detail)
   - Safe Delete Candidates (filtered view)
7. Optionally **emails the report** to you and your boss
8. Allows deletion **only** after you manually mark rows `ApprovedToDelete=Yes` and pass the `--delete-approved` flag

---

## Why This Is Safe

- Read-only by default — no Azure resources are modified on a normal run
- Deletion requires TWO layers of protection: the flag AND the CSV column AND a typed CONFIRM prompt
- No credentials are stored anywhere — uses the active `az login` session
- Terraform-managed endpoints are automatically flagged DO NOT DELETE
- Endpoints with live backend resources are flagged INVESTIGATE

---

## Prerequisites (Install These First)

### 1. Azure CLI
Download and install from: https://aka.ms/installazurecliwindows

Verify it works:
```
az --version
```

### 2. Python 3.10+
Download from: https://www.python.org/downloads/

Verify:
```
python --version
```

### 3. Git (optional, for cloning)
Download from: https://git-scm.com/download/win

---

## Setup — Do This Once

### Step 1: Get the code

Option A — Clone (if you have Git):
```
git clone https://github.com/ausjones84/edav-private-endpoint-monitor.git
cd edav-private-endpoint-monitor
```

Option B — Download ZIP from GitHub, unzip it, open a PowerShell window inside the folder.

### Step 2: Create a virtual environment
```
python -m venv .venv
.venv\Scripts\activate
```

You should see `(.venv)` appear at the start of your prompt.

### Step 3: Install dependencies
```
pip install -r requirements.txt
```

### Step 4: Login to Azure with your SU account
```
az login --use-device-code
```
It will give you a code. Go to https://microsoft.com/devicelogin, enter the code,
and sign in with your CDC SU account.

Then set the subscription you want to scan:
```
az account set --subscription "OCIO-TSBDEV-C1"
```

Verify you are in:
```
az account show
```

---

## How To Run

### Basic scan (report only — safest option)
```
python main.py --input "C:\Users\bh55\Downloads\EDAV_Disconnected_Private_Endpoints_Full_Report.csv" --subscriptions "OCIO-TSBDEV-C1"
```

### Multi-subscription scan (recommended — covers all your environments)
```
python main.py --input "C:\Users\bh55\Downloads\EDAV_Disconnected_Private_Endpoints_Full_Report.csv" --subscriptions "OCIO-TSBDEV-C1,OCIO-TSBPRD-C1,OCIO-EDAV-DMZ-C1,OCIO-DMZ-C1"
```

### With Terraform check (best if you have the repo cloned locally)
```
python main.py --input "C:\Users\bh55\Downloads\EDAV_Disconnected_Private_Endpoints_Full_Report.csv" --subscriptions "OCIO-TSBDEV-C1,OCIO-TSBPRD-C1,OCIO-EDAV-DMZ-C1,OCIO-DMZ-C1" --terraform-path "C:\Users\bh55\terraform-scripts"
```

### With email report sent to your boss
```
python main.py --input "C:\Users\bh55\Downloads\EDAV_Disconnected_Private_Endpoints_Full_Report.csv" --subscriptions "OCIO-TSBDEV-C1,OCIO-TSBPRD-C1" --email-to "boss@cdc.gov,you@cdc.gov" --email-from "you@cdc.gov" --smtp-server "smtp.cdc.gov" --smtp-port 587
```

---

## What You Will See

While running, it prints status for each endpoint:
```
2026-04-29 14:23:01  INFO     [1/102] testwebbseries-pe
2026-04-29 14:23:03  INFO     [2/102] tempendpoint
...
============================================================
SUMMARY  (Total: 102)
============================================================
  Safe Delete Candidate                              23
  Endpoint Not Found / Check Subscription or Access 45
  Investigate - Backend Exists                       18
  Do Not Delete - Terraform Managed                  6
  Review                                             10
============================================================
```

It saves two files to the `reports/` folder:
- `EDAV_Validation_Report_YYYYMMDD_HHMMSS.xlsx` — colour-coded Excel
- `EDAV_Validation_Report_YYYYMMDD_HHMMSS.csv` — plain CSV
- `summary_YYYYMMDD_HHMMSS.md` — markdown summary

---

## Understanding the Excel Report

| Colour | Recommended Action | What To Do |
|---|---|---|
| GREEN | Safe Delete Candidate | These are safe to decommission after change ticket approval |
| RED | Do Not Delete - Terraform Managed | Do NOT touch — must be removed from Terraform code first |
| YELLOW | Investigate - Backend Exists | Endpoint is disconnected but backend resource still exists — review |
| GREY | Endpoint Not Found | Not found in scanned subscription — try another subscription |

---

## How To Approve and Delete (After Change Ticket Approval)

### Step 1: Open the Excel report
Go to the `Safe Delete Candidates` tab.

### Step 2: Mark rows approved
In the main CSV report, add `Yes` in the `ApprovedToDelete` column for each endpoint
your change ticket has approved for deletion. Save as CSV.

### Step 3: Re-run with delete flag
```
python main.py --input "reports\EDAV_Validation_Report_YYYYMMDD_HHMMSS.csv" --subscriptions "OCIO-TSBDEV-C1" --delete-approved
```

It will show you how many endpoints are queued and ask you to type `CONFIRM` before anything is deleted.

**Do NOT use --delete-approved without an approved change ticket.**

---

## Colour Code Quick Reference

```
GREEN  = Safe Delete Candidate    --> OK to decommission (after approval)
RED    = Terraform Managed        --> DO NOT DELETE manually
YELLOW = Investigate              --> Needs human review
GREY   = Not Found / Unknown      --> Wrong subscription or access issue
```

---

## What To Say to Your Boss / In Standup

```
I built a repeatable validation pipeline for the EDAV disconnected private endpoint cleanup.
The tool reads the EDAV CSV, scans Azure using my SU account, validates each endpoint
against its backend resource, checks Terraform ownership to prevent recreation issues,
and generates a colour-coded report showing what is safe to decommission.
Deletion is gated behind change ticket approval - nothing is removed automatically.
I can schedule this to run weekly and email the report automatically.
```

---

## Storing Reports in Azure (Optional)

After you generate a report locally, you can upload it to Azure Blob Storage:
```
az storage blob upload --account-name <your-storage-account> --container-name edav-reports --file "reports\EDAV_Validation_Report_YYYYMMDD.xlsx" --name "EDAV_Validation_Report_YYYYMMDD.xlsx" --auth-mode login
```

This way the report lives in Azure and your boss can access it from there.
No extra Azure resources needed — just an existing storage account.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `az: command not found` | Install Azure CLI from aka.ms/installazurecliwindows |
| `Not logged in` | Run `az login --use-device-code` |
| `ModuleNotFoundError: pandas` | Run `pip install -r requirements.txt` |
| All endpoints say `Endpoint Not Found` | You are on the wrong subscription. Run `az account list -o table` and switch subscriptions |
| `Terraform not found` | Skip `--terraform-path` — Terraform checks will show Unknown |
| Can not access EDAV dashboard | Make sure Zscaler Client Connector is ON and connected with CDC creds |

---

## File Structure

```
edav-private-endpoint-monitor/
|-- main.py                 <- The main script (run this)
|-- requirements.txt        <- Python packages to install
|-- sample_input.csv        <- Example input file format
|-- .gitignore              <- Keeps reports and data files out of git
|-- README.md               <- This file
|-- reports/                <- Output folder (reports saved here)
```

---

## Author
Built by Austin Jones for EDAV infrastructure cost cleanup at CDC/OCIO.

**Version:** 2.0.0
**Safe:** Read-only by default. Approval-gated deletion only.
