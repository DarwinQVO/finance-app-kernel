# 1. Upload and Process Bank Statements

**Goal:** Upload bank PDFs and extract transaction data

---

## What the user needs

**Upload bank statements:**
- Drag & drop PDF files from Bank of America
- See upload progress
- Get confirmation when upload completes

**Automatic processing:**
- System reads PDF and extracts transactions
- System cleans up dates, amounts, descriptions
- No manual data entry required

**Handle errors:**
- If PDF can't be read → show error message
- If duplicate file uploaded → tell user "already uploaded"
- Retry failed uploads

---

## Scale expectations

- **Volume:** ~10 PDFs per month (one per account)
- **File size:** 100KB - 2MB per PDF
- **Transactions:** ~500 transactions/month total
- **Processing time:** Under 30 seconds per PDF
- **Storage:** ~500MB per year

---

## User experience

### Upload flow

1. **User clicks "Upload Statement"**
   - Opens file picker
   - Selects one or more PDF files

2. **Upload happens**
   - Progress bar shows upload status
   - "Processing..." indicator appears

3. **Results shown**
   - Success: "42 transactions imported from statement.pdf"
   - Error: "Could not read statement.pdf - file may be corrupted"
   - Duplicate: "This file was already uploaded on Jan 15"

### What happens behind the scenes

1. File uploaded to storage
2. Parser extracts raw transaction data
3. Data cleaner normalizes dates, amounts, descriptions
4. Transactions saved to database
5. User can now view transactions

---

## Technical constraints (simple)

**Storage:**
- Local filesystem is fine (not S3, not cloud)
- Keep original PDFs for reference

**Database:**
- SQLite is sufficient for 500 transactions/month
- No need for PostgreSQL or sharding

**Parser:**
- Simple PDF table extraction (PyPDF2 or similar)
- No machine learning required
- No OCR required (PDFs are digital, not scanned)

**Processing:**
- Synchronous processing is OK (user waits 30 seconds)
- No need for background workers or queues initially
- Can add async processing later if needed

---

## What's NOT needed (out of scope)

❌ Upload from mobile app (web only)
❌ Automatic bank sync (upload only)
❌ Support for other banks (BoFA only initially)
❌ CSV import (PDF only initially)
❌ Batch upload of 100+ files
❌ Real-time streaming imports
❌ Multi-user file sharing
