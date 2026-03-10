# SQL Data Validation Queries — SAP MM & SD

**Eluru Priyanka | QA Portfolio**  
These queries are used for backend data validation during SAP functional and integration testing.

---

## SAP MM Queries

### SQL-MM-001 — Verify Purchase Order Header Data
**Purpose:** Validate PO was saved correctly with correct vendor, currency, and net value.  
**SAP Table:** `EKKO` (Purchase Order Header)

```sql
SELECT EBELN, BUKRS, BSTYP, LIFNR, EKORG, WAERS, NETWR, BEDAT
FROM EKKO
WHERE EBELN = '4500067890'
  AND BUKRS = '1000';
```
**Expected Output:** 1 row — correct vendor (LIFNR), PO date (BEDAT), currency (WAERS), net value (NETWR).

---

### SQL-MM-002 — Validate PO Line Items
**Purpose:** Confirm each PO line item has correct material, quantity, unit, price, and plant.  
**SAP Table:** `EKPO` (Purchase Order Item)

```sql
SELECT EBELN, EBELP, MATNR, MENGE, MEINS, NETPR, WERKS
FROM EKPO
WHERE EBELN = '4500067890'
ORDER BY EBELP;
```
**Expected Output:** Rows matching test data — MATNR, MENGE, MEINS, NETPR, WERKS all correct.

---

### SQL-MM-003 — Confirm Goods Receipt Document Exists
**Purpose:** Verify GR was posted and material document created.  
**SAP Table:** `MKPF` (Material Document Header)

```sql
SELECT MBLNR, MJAHR, BUDAT, MANDT, BLART
FROM MKPF
WHERE MBLNR = '5000011111'
  AND MJAHR = '2023';
```
**Expected Output:** 1 row — document type `WE` (Goods Receipt), correct posting date.

---

### SQL-MM-004 — Validate Stock Updated After GR
**Purpose:** Confirm unrestricted stock (LABST) increased by GR quantity.  
**SAP Table:** `MARD` (Storage Location Stock)

```sql
SELECT MATNR, WERKS, LGORT, LABST, UMLME, INSME
FROM MARD
WHERE MATNR = 'RM-001'
  AND WERKS = '1000'
  AND LGORT = '0001';
```
**Expected Output:** `LABST` increased by 100 units after GR posting.

---

### SQL-MM-005 — Check Invoice Verification (LIV) Document
**Purpose:** Confirm 3-way match invoice document posted correctly.  
**SAP Table:** `RBKP` (Invoice Document Header)

```sql
SELECT BELNR, BUKRS, BLDAT, BUDAT, LIFNR, DMBTR, WAERS
FROM RBKP
WHERE BELNR = '5105000222'
  AND BUKRS = '1000';
```
**Expected Output:** Invoice header with vendor, amount (DMBTR) matching PO net value, and posting date.

---

## SAP SD Queries

### SQL-SD-001 — Verify Sales Order Header
**Purpose:** Confirm SO created with correct sales org, customer, net value, and date.  
**SAP Table:** `VBAK` (Sales Order Header)

```sql
SELECT VBELN, VKORG, VTWEG, SPART, KUNNR, NETWR, WAERK, ERDAT
FROM VBAK
WHERE VBELN = '1000013456';
```
**Expected Output:** SO with correct VKORG, KUNNR = C-2001, NETWR, and creation date.

---

### SQL-SD-002 — Validate SO Line Items & Pricing
**Purpose:** Confirm material, quantity, unit, and net price per item are correct.  
**SAP Table:** `VBAP` (Sales Order Item)

```sql
SELECT VBELN, POSNR, MATNR, KWMENG, VRKME, NETPR, WERKS
FROM VBAP
WHERE VBELN = '1000013456'
ORDER BY POSNR;
```
**Expected Output:** Each item shows correct MATNR, confirmed qty (KWMENG), and net price (NETPR).

---

### SQL-SD-003 — Check Outbound Delivery
**Purpose:** Verify delivery was created with correct ship-to and shipping point.  
**SAP Tables:** `LIKP` (Delivery Header), `LIPS` (Delivery Items)

```sql
-- Header
SELECT VBELN, VSTEL, KUNNR, WADAT_IST, LFART
FROM LIKP
WHERE VBELN = '8000054321';

-- Items
SELECT VBELN, POSNR, MATNR, LFIMG, VRKME
FROM LIPS
WHERE VBELN = '8000054321';
```
**Expected Output:** Correct ship-to (KUNNR), shipping point (VSTEL), and actual delivery qty (LFIMG).

---

### SQL-SD-004 — Confirm Stock Reduced After PGI
**Purpose:** Validates SD-MM integration — stock reduced after Goods Issue.  
**SAP Table:** `MARD` (Storage Location Stock)

```sql
SELECT MATNR, WERKS, LABST
FROM MARD
WHERE MATNR = 'FG-001'
  AND WERKS = '1000'
  AND LGORT = '0001';
```
**Expected Output:** `LABST` reduced by 50 units after PGI. Critical SD↔MM integration check.

---

### SQL-SD-005 — Validate Billing Document & FI Accounting Entry
**Purpose:** Confirm billing document created and FI accounting entry posted.  
**SAP Tables:** `VBRK` (Billing Header), `BKPF` (FI Document Header)

```sql
-- Billing Document
SELECT VBELN, FKART, KUNAG, NETWR, WAERK, FKDAT
FROM VBRK
WHERE VBELN = '9000098765';

-- FI Accounting Document
SELECT BELNR, BUKRS, BUDAT, DMBTR
FROM BKPF
WHERE AWKEY = '9000098765'
  AND BUKRS = '1000';
```
**Expected Output:** Billing type `F2`, correct customer and amount. FI document confirming accounting entry.

---

## Integration Queries

### SQL-INT-001 — End-to-End O2C Document Flow Validation
**Purpose:** Validate full Order-to-Cash document chain (SO → Delivery → GI → Invoice).  
**SAP Table:** `VBFA` (Document Flow)

```sql
SELECT VBFA.VBELV AS Source_Doc,
       VBFA.VBELN AS Target_Doc,
       VBFA.VBTYP_N AS Target_Type,
       VBFA.RFMNG AS Qty
FROM VBFA
WHERE VBFA.VBELV = '1000013456'
ORDER BY VBFA.VBTYP_N;
```
**Expected Output:** Document chain: SO → Delivery (J) → GI (R) → Invoice (M). All types present confirms complete O2C flow.

---

### SQL-INT-002 — Find All Open POs With Pending GR
**Purpose:** List all POs with outstanding GR quantity for reconciliation and regression validation.  
**SAP Tables:** `EKKO`, `EKPO`, `MSEG`

```sql
SELECT E.EBELN,
       E.LIFNR,
       P.MATNR,
       P.MENGE                          AS PO_Qty,
       COALESCE(SUM(M.MENGE), 0)        AS GR_Qty,
       P.MENGE - COALESCE(SUM(M.MENGE), 0) AS Pending_Qty
FROM EKKO E
JOIN EKPO P ON E.EBELN = P.EBELN
LEFT JOIN MSEG M
       ON M.EBELN = E.EBELN
      AND M.EBELP = P.EBELP
      AND M.BEWTP = 'E'
WHERE E.LOEKZ = ''
  AND P.LOEKZ = ''
GROUP BY E.EBELN, E.LIFNR, P.MATNR, P.MENGE
HAVING P.MENGE > COALESCE(SUM(M.MENGE), 0)
ORDER BY E.EBELN;
```
**Expected Output:** All POs with outstanding GR quantity. Used for data migration and regression validation.

---

*These queries target SAP ECC 6.0 database tables. Column names and table structures follow standard SAP naming conventions.*
