# n8n Workflow Refactoring - NODO to WolfCRM Integration

## Overview

The n8n workflow has been completely refactored to fix the critical issue where documents could not be uploaded during or after contract creation. The workflow now uploads all documents **WITH** the contract in a single API call, preventing issues with contracts being frozen during the digital signature process.

## Critical Problem Fixed

**Before**: Documents were being uploaded AFTER contract creation, which failed because:
- WolfCRM contracts become "frozen" once they enter the digital signature workflow
- The API endpoint for updating contracts with documents was not working as expected
- Documents could not be attached retroactively

**After**: Documents are now downloaded and processed BEFORE creating the contract, then attached during the initial `setContract` API call. This ensures all documents are included from the start.

## Major Changes

### 1. Workflow Restructuring

**New node execution order**:
```
1. Webhook/Manual Trigger
2. Get Contract from NODO
3. Authenticate with WolfCRM
4. Fetch reference data (counties, products)
5. Process customer (check existence, create if needed)
6. **NEW: Check if documents exist**
7. **NEW: Download ALL documents from NODO**
8. **NEW: Map documents to WolfCRM field names**
9. **MODIFIED: Create contract WITH files attached**
10. Process result
```

### 2. New Nodes Added

- **"Has Documents?" (IF node)**: Checks if the NODO contract includes documents
- **"Split Documents"**: Iterates over each document in the array
- **"Download Document"**: Downloads binary file from NODO API
- **"Map Documents to Wolf Fields"**: Intelligent mapping of NODO document types to WolfCRM fields
- **"Merge With Documents"**: Combines downloaded files with contract data
- **"Merge Customer Paths"**: Ensures customer data flows correctly from both branches

### 3. Document Type Mapping Logic

The workflow now intelligently maps NODO document types to WolfCRM fields:

| NODO Document Type/Filename | WolfCRM Field |
|------------------------------|---------------|
| CIF, DNI, NIE (type or in filename) | VAT_FILE |
| FACTURA, INVOICE (in filename) | INVOICE_FILE |
| ANEXO, CONTRATO (in filename) | ANEXO_RESIDENCIAL_FILE |
| All others | OTHER_FILE |

### 4. Modified Nodes

**"Create Contract" → "Create Contract with Files"**:
- Added binary file parameters for VAT_FILE, INVOICE_FILE, ANEXO_RESIDENCIAL_FILE, OTHER_FILE
- Uses dynamic expressions to only send files that are available
- Maintains all original contract field mappings

### 5. Removed Nodes

- All post-creation document upload logic
- Obsolete document processing nodes that ran after contract creation

## Technical Details

### File Upload Implementation

The contract creation now uses `multipart/form-data` with binary attachments:

```javascript
// Dynamic file attachment based on available documents
VAT_FILE: {{ $json.mapped_fields?.includes('VAT_FILE') ? 'VAT_FILE' : '' }}
INVOICE_FILE: {{ $json.mapped_fields?.includes('INVOICE_FILE') ? 'INVOICE_FILE' : '' }}
ANEXO_RESIDENCIAL_FILE: {{ $json.mapped_fields?.includes('ANEXO_RESIDENCIAL_FILE') ? 'ANEXO_RESIDENCIAL_FILE' : '' }}
OTHER_FILE: {{ $json.mapped_fields?.includes('OTHER_FILE') ? 'OTHER_FILE' : '' }}
```

### Binary Data Flow

1. Documents downloaded from NODO as binary data
2. Each document stored in n8n binary storage with unique key
3. Mapping function groups documents by WolfCRM field name
4. Contract creation node references binary data by field name
5. All files sent in single HTTP request

## Testing Instructions

### Prerequisites

1. Import the updated workflow into your n8n instance
2. Configure WolfCRM credentials (already in `Credenciales.txt`)
3. Ensure NODO API access is working

### Test Environment

Use the test credentials from `Credenciales.txt`:
```
Host: https://canallogos.v3.dev.partnertec.es/application/ws/v1/index.php
ClientId: EnergiaMia2020
Secret: babf843e78a37b3caed2a8882291a674bfa1ec77a1be07e5f1c779c5d4269792
```

### Test Cases

#### Test Case 1: Contract with Documents
1. Trigger workflow with a NODO contract ID that has documents attached
2. Monitor execution flow - ensure it goes through document download path
3. Verify contract is created in WolfCRM
4. Check WolfCRM contract detail page - verify all documents are attached
5. Expected: All documents present, correctly categorized by type

#### Test Case 2: Contract without Documents
1. Trigger workflow with a NODO contract ID that has NO documents
2. Monitor execution flow - should bypass document download
3. Verify contract is created successfully
4. Expected: Contract created without errors, no documents attached

#### Test Case 3: Multiple Document Types
1. Use a NODO contract with DNI, FACTURA, and ANEXO documents
2. Verify each document maps to correct WolfCRM field:
   - DNI → VAT_FILE
   - FACTURA → INVOICE_FILE
   - ANEXO → ANEXO_RESIDENCIAL_FILE
3. Expected: All three documents attached to correct fields

#### Test Case 4: Customer Creation + Documents
1. Use a new customer not in WolfCRM with documents
2. Verify customer is created first
3. Verify contract is created with customer ID
4. Verify documents are attached
5. Expected: Customer created, contract created, documents attached - all in one workflow run

### Monitoring Points

Watch these workflow nodes during testing:

1. **"Has Documents?"**: Should correctly detect presence of documents
2. **"Download Document"**: Should complete without errors for each document
3. **"Map Documents to Wolf Fields"**: Check the output to verify correct mapping
4. **"Create Contract with Files"**:
   - Should return HTTP 200
   - Response should include contract objectId
   - No error messages about files

### Common Issues to Check

- **401 Unauthorized**: Token may have expired (60-minute validity), re-run auth
- **400 Bad Request with file errors**: Check binary data is properly formatted
- **Missing documents in WolfCRM**: Verify mapping logic executed correctly
- **Contract created but no documents**: Check that document binary data was available in merge step

## Rollback Plan

If issues occur, the original workflow has been preserved:
```
File: n8n workflow.backup-20260128-115225
```

To rollback:
1. Copy backup file over current workflow file
2. Re-import into n8n
3. Report issues for investigation

## Files Modified

- `n8n workflow` - Main workflow file (COMPLETELY REFACTORED)
- `n8n workflow.backup-20260128-115225` - Backup of original (NEW)

## Git History

```
9e5fc9b - Add backup of original workflow before refactoring
c70946c - Refactor n8n workflow to upload documents during contract creation
```

## Next Steps

1. **Test thoroughly** in the test environment using all test cases above
2. **Monitor for errors** - especially around document attachment
3. **Verify in WolfCRM UI** that documents appear correctly
4. **Once validated**, update credentials to production environment
5. **Run production tests** with real NODO contracts
6. **Monitor production** for any edge cases not covered in testing

## Support

If you encounter issues:
1. Check n8n execution logs for specific error messages
2. Verify WolfCRM API response codes and messages
3. Review the document mapping output to ensure correct field assignment
4. Check that NODO documents are accessible and downloadable

## API Reference

- **WolfCRM API Docs**: See `Logos - EnergiaMia2020 - WolfCRM.postman_collection.json`
- **Official Docs**: See `PRUEBAS_WolfCRM_Ws_V1.pdf` (test) and `PRODUCCION_WolfCRM_Ws_V1.pdf` (production)
- **Credentials**: See `Credenciales.txt`

---

**Implementation Date**: 2026-01-28
**Implemented By**: Claude Code
**Branch**: `claude/nodo-wolfcrm-workflow-4v1BR`
