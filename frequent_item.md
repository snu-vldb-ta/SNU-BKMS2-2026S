# Frequent Itemset Counting in Oracle — Lab Setup Guide

This guide walks you through setting up the Oracle SH (Sales History) schema on your Docker container and running the Frequent Itemset Counting script.

---

## Prerequisites

Your Oracle 26ai Free container should already be running:

```bash
docker run -d --name oracle-free \
  -p 1521:1521 \
  -p 5500:5500 \
  -e ORACLE_PWD="1234" \
  container-registry.oracle.com/database/free:latest
```

Wait 1–2 minutes for the database to start. You can check readiness with:

```bash
docker logs oracle-free
```

Look for `DATABASE IS READY TO USE!` in the output.

---

## Step 1: Install SQLcl on Your Host PC

The SH schema requires **SQLcl** (not SQL\*Plus) to load data. Install it on your **host machine** (not inside the container).

### macOS

```bash
brew install sqlcl
```

Verify:
```bash
sql -v
```

### Windows

1. Download SQLcl from: https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/download/
2. Unzip the downloaded file (e.g., to `C:\sqlcl`)
3. Add `C:\sqlcl\bin` to your system PATH:
   - Search "Environment Variables" in Windows Settings
   - Edit the `Path` variable under "System variables"
   - Add `C:\sqlcl\bin`
4. Open a **new** Command Prompt or PowerShell and verify:
   ```
   sql -v
   ```

> **Note**: SQLcl requires Java 11+. If you get a Java error, install Java from https://adoptium.net/

---

## Step 2: Download the Sample Schemas

Run these commands on your **host machine** (not inside the container).

### macOS / Linux

```bash
cd ~/Downloads
curl -L -o schemas.zip https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v23.3.zip
unzip schemas.zip
cd db-sample-schemas-23.3/sales_history
```

### Windows (PowerShell)

```powershell
cd $HOME\Downloads
Invoke-WebRequest -Uri "https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v23.3.zip" -OutFile schemas.zip
Expand-Archive schemas.zip -DestinationPath .
cd db-sample-schemas-23.3\sales_history
```

### Windows (Command Prompt)

Or simply download the zip manually from your browser:
1. Go to https://github.com/oracle-samples/db-sample-schemas/archive/refs/tags/v23.3.zip
2. Unzip the downloaded file
3. Open Command Prompt and navigate to the `sales_history` folder:
   ```
   cd %USERPROFILE%\Downloads\db-sample-schemas-23.3\sales_history
   ```

---

## Step 3: Install the SH Schema

From the `sales_history` directory on your host machine, run:

```bash
sql system/1234@//localhost:1521/FREEPDB1 @sh_install.sql
```

You will be prompted for three inputs:

| Prompt | What to enter |
|--------|---------------|
| Enter a password for the user SH | `sh` |
| Enter a tablespace for SH | Just press **Enter** (uses default) |
| Do you want to overwrite the schema? | `yes` |

Wait for the installation to complete. At the end, you should see a verification table like this:

```
Installation verification
___________________________
Verification:
  Table         provided    actual
  channels      5           5
  costs         82112       82112
  countries     35          35
  customers     55500       55500
  products      72          72
  promotions    503         503
  sales         918843      918843
  times         1826        1826
```

If all numbers match, the installation was successful.

---

## Step 4: Verify the Installation

Enter the container and connect as SH:

```bash
docker exec -it oracle-free bash
```

```bash
sqlplus sh/sh@//localhost:1521/FREEPDB1
```

Run these checks:

```sql
SELECT COUNT(*) FROM products;
SELECT COUNT(*) FROM sales;
SELECT COUNT(DISTINCT cust_id) FROM sales;
```

Expected results:
- products: **72**
- sales: **918843**
- distinct cust_id: **7059**

If any count is **0**, the installation failed — go back to Step 3.

Type `EXIT;` to leave SQL\*Plus when done checking.

---

## Step 5: Run the Frequent Itemset Counting Script

Connect to SH again (if you exited):

```bash
sqlplus sh/sh@//localhost:1521/FREEPDB1
```

Run the following commands **one section at a time** at the `SQL>` prompt.

### 5-1. Check the Schema

```sql
DESC sales;
DESC products;

SELECT prod_name FROM products;

SELECT AVG(cnt)
FROM (SELECT cust_id, COUNT(*) cnt
      FROM sales
      GROUP BY cust_id);
```

### 5-2. Frequent Itemsets with Numeric Item IDs

```sql
CREATE OR REPLACE TYPE fi_num AS TABLE OF NUMBER;
/

SET LINESIZE 100;
COLUMN ITEMSET FORMAT A40;

SELECT CAST(itemset AS fi_num) itemset, support, length, total_tranx
FROM TABLE(DBMS_FREQUENT_ITEMSET.FI_TRANSACTIONAL(
               CURSOR(SELECT cust_id, prod_id FROM sales),
               (5000/7059),
               2,
               5,
               NULL,
               NULL)
     )
/
```

### 5-3. Frequent Itemsets with Product Names

```sql
CREATE OR REPLACE TYPE fi_char AS TABLE OF VARCHAR2(100);
/

SET LINES 200;
COLUMN ITEMSET FORMAT A120;
COLUMN SUPPORT FORMAT 9,999,999;
COLUMN "LENGTH" FORMAT 999,999;
COLUMN total_tranx FORMAT 999,999;

SELECT CAST(itemset AS fi_char) itemset, support, length, total_tranx
FROM TABLE(DBMS_FREQUENT_ITEMSET.FI_TRANSACTIONAL(
               CURSOR(SELECT s.cust_id, p.prod_name
                      FROM sales s, products p
                      WHERE s.prod_id = p.prod_id),
               (5000/7059),
               2,
               5,
               NULL,
               NULL)
     )
/
```

### 5-4. Experiment: Lower the Support Threshold

Try changing `5000/7059` to `4000/7059` to discover more frequent itemsets:

```sql
SELECT CAST(itemset AS fi_char) itemset, support, length, total_tranx
FROM TABLE(DBMS_FREQUENT_ITEMSET.FI_TRANSACTIONAL(
               CURSOR(SELECT s.cust_id, p.prod_name
                      FROM sales s, products p
                      WHERE s.prod_id = p.prod_id),
               (4000/7059),
               2,
               5,
               NULL,
               NULL)
     )
/
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `sql` command not found | SQLcl not installed or not in PATH | See Step 1 |
| `ORA-01017: invalid username/password` | SH user not created | Re-run Step 3 |
| `ORA-00942: table or view does not exist` | Tables not created | Re-run Step 3 |
| All table counts are **0** | Data not loaded (used SQL\*Plus instead of SQLcl) | Must use `sql` (SQLcl), not `sqlplus` in Step 3 |
| `no rows selected` on itemset query | Support threshold too high | Lower it (e.g., `3000/7059`) |
| `bash: command not found` with weird characters | Korean input method active | Switch to English input, type manually |
| Container not starting | Docker not running | Start Docker Desktop first |
| `Cannot connect to database` from host | Container not ready | Wait 1–2 min, check `docker logs oracle-free` |
| Java error when running SQLcl | Java not installed or too old | Install Java 11+ from https://adoptium.net/ |

---

## Quick Reference

```bash
# Start the container (if stopped)
docker start oracle-free

# Enter the container
docker exec -it oracle-free bash

# Connect as SH (inside container)
sqlplus sh/sh@//localhost:1521/FREEPDB1

# Connect as SYSTEM (inside container)
sqlplus system/1234@//localhost:1521/FREEPDB1

# Exit SQL*Plus
EXIT;

# Exit the container shell
exit
```
