# Document Template Module Analysis - PowerSuite

## Executive Summary

The PowerSuite codebase has a comprehensive document template system that manages headers and footers for multiple document types (invoices, costs, receipts, payments, deposits, credit notes, debit notes). The system uses a branch-level template approach, with fallback to company-level data.

**Key Finding**: There is NO existing WYSIWYG text editor for template management. Templates are currently edited via plain text form fields (Textarea components).

---

## 1. Document Template Architecture

### 1.1 Database Schema

**Model**: `/mnt/c/Codes/Powersuite/psback/models/system_models/branch_document_template.js`

```javascript
branch_document_template {
  id: INTEGER (primary key)
  branch_id: INTEGER (FK to branches)
  company_code: STRING (FK to companies)
  document_type: ENUM('invoice', 'cost', 'receipt', 'payment', 'deposit', 'credit_note', 'debit_note')
  document_title: STRING
  header_content: MEDIUMTEXT (HTML, WYSIWYG)
  footer_content: MEDIUMTEXT (HTML, WYSIWYG)
  stamp_image: MEDIUMTEXT (base64 stamp image, shown above Authorized Signature on invoice)
  stamp_opacity: INTEGER (0-100, default 55) - user-set transparency of the stamp
  stamp_width: INTEGER (px, optional) - fixed stamp width, blank = auto
  stamp_height: INTEGER (px, optional) - fixed stamp height, blank = auto
  header_name: STRING
  header_address: TEXT
  header_phone: STRING(100)
  header_email: STRING
  header_licence_no: STRING(100)
  header_ntn: STRING(100)
  footer_text: TEXT
  created_at: TIMESTAMP
  updated_at: TIMESTAMP
  
  Constraints:
  - UNIQUE(branch_id, document_type) - One template per document type per branch
  - Indexes on branch_id and company_code
}
```

**Related Model**: `/mnt/c/Codes/Powersuite/psback/models/invoice_footer.js`
```javascript
invoice_footer {
  id: INTEGER (primary key)
  invoice_footer: TEXT (legacy/alternative footer storage)
  timestamps: true
}
```

**Note**: The `invoice_footer` model appears to be legacy. The current system uses `branch_document_template.footer_text`.

### 1.1.1 Stamp (Signature Stamp)

A stamp can be uploaded per branch + document type from the template editor
(`EditBranchDocumentTemplate.jsx`, **Stamp** section, located between Header and Footer).

- The image is stored as a base64 data URL in `stamp_image`.
- The user controls `stamp_opacity` (0-100%), and optional `stamp_width` / `stamp_height` in px (blank = auto).
- On the **invoice** print (`invoiceDocument.ejs`), the stamp renders just above the
  "Authorized Signature" line, overlapping it slightly (negative bottom margin) at the
  chosen opacity, so both the stamp and the signature text stay readable.
- Only the invoice document renders the stamp today. Other document types store the
  fields but do not print the stamp unless their EJS is extended.

### 1.2 Supported Document Types & Mapping

```javascript
DOCUMENT_TYPES = [
  { type: 'invoice', title: 'TAX INVOICE' },
  { type: 'cost', title: 'COST DOCUMENT' },
  { type: 'receipt', title: 'RECEIPT SETTLEMENT' },
  { type: 'payment', title: 'PAYMENT SETTLEMENT' },
  { type: 'deposit', title: 'CUSTOMER DEPOSIT' },
  { type: 'credit_note', title: 'CREDIT NOTE' },
  { type: 'debit_note', title: 'DEBIT NOTE' }
];
```

**Type Mapping** (`mapDocumentTypeToTemplateType` in `document.controller.js`):
- 'invoice' → 'invoice'
- 'costing' → 'cost'
- 'payment' → 'payment'
- 'receipt' → 'receipt'
- 'deposit' → 'deposit'
- 'credit_note' → 'credit_note'
- 'debit_note' → 'debit_note'

---

## 2. Backend Implementation

### 2.1 Service Layer

**File**: `/mnt/c/Codes/Powersuite/psback/services/branch_document_template.service.js`

**Key Functions**:

1. **`createDefaultTemplatesForBranch(branchId, companyCode)`**
   - Creates 7 default templates (one per document type) for a branch
   - Populates header fields from company data
   - Generates default footer text from company info
   - Used during branch creation or manual initialization

2. **`getTemplateByBranchAndType(branchId, documentType)`**
   - Retrieves specific template
   - Returns null if not found

3. **`getTemplatesByBranch(branchId)`**
   - Gets all templates for a branch
   - Orders by document_type ASC

4. **`updateTemplate(templateId, updateData)`**
   - Updates allowed fields: document_title, header_name, header_address, header_phone, header_email, header_licence_no, header_ntn, footer_text
   - Cannot update: branch_id, company_code, document_type (immutable)

5. **`updateTemplateByBranchAndType(branchId, documentType, updateData)`**
   - Convenience wrapper around updateTemplate

6. **`deleteTemplatesByBranch(branchId)`**
   - Cascades when branch is deleted

### 2.2 Controller Layer

**File**: `/mnt/c/Codes/Powersuite/psback/controllers/branch_document_template.controller.js`

**Endpoints**:
```
GET    /branch-document-template/branch/:branchId
GET    /branch-document-template/branch/:branchId/type/:documentType
PUT    /branch-document-template/:id
PUT    /branch-document-template/branch/:branchId/type/:documentType
POST   /branch-document-template/branch/:branchId/initialize
```

### 2.3 Document Rendering with Templates

**File**: `/mnt/c/Codes/Powersuite/psback/controllers/document.controller.js`

**Key Function**: Document rendering with template fallback

```javascript
// Lines 1190-1210: Invoice document rendering
exports.getInvoiceDocument = async (req, res) => {
  // ... fetch invoice data ...
  
  if (branch && branch.id) {
    branchTemplate = await branchTemplateService.getTemplateByBranchAndType(
      branch.id, 
      'invoice'
    );
  }
  
  return res.render("pages/invoiceDocument", {
    invoices,
    template: branchTemplate  // Pass template to view
    // ... other data ...
  });
};
```

**Functions for Headers/Footers**:

1. **`getDocumentHeader(documentNumber)`** (Lines 1231-1325)
   - Extracts branch prefix from document number (e.g., "TTIN00000084" → "TT")
   - Looks up branch by prefix
   - Fetches template by branch ID and document type
   - Renders `/pages/invoiceHeader` with template data
   - Fallback: Uses company data if no template found

2. **`getDocumentFooter(documentNumber)`** (Lines 1327-1417)
   - Same logic as header but for footer
   - Renders `/pages/invoiceFooter` with template data

**Branch Prefix Extraction Logic**:
```javascript
const docTypePrefixes = ['IN', 'XO', 'RP', 'DP', 'PY', 'CN', 'DN'];
// IN = Invoice, XO = Cost, RP = Receipt, DP = Deposit, 
// PY = Payment, CN = Credit Note, DN = Debit Note
```

---

## 3. EJS View Templates

### 3.1 Header Template

**File**: `/mnt/c/Codes/Powersuite/psback/views/pages/invoiceHeader.ejs`

```ejs
<div class="header">
  <div class="header--logo-text">
    <%= template?.header_name || company?.name || '' %>
  </div>
  <div class="logo--address">
    <%= template?.header_address || company?.address || '' %>
    <br>
    Phone: <%= template?.header_phone || company?.phone || '' %>
    <% if (template?.header_email || company?.email) { %>
      <br>
      Email: <%= template?.header_email || company?.email || '' %>
    <% } %>
    <br>
    LICENSE NO: <%= template?.header_licence_no || company?.licence_no || '' %>, 
    NTN: <%= template?.header_ntn || company?.ntn || '' %>
    <br>
    <strong><%= template?.document_title || 'TAX INVOICE' %></strong>
  </div>
</div>
```

**Template Variables Used**:
- `template?.header_name` (fallback: `company?.name`)
- `template?.header_address` (fallback: `company?.address`)
- `template?.header_phone` (fallback: `company?.phone`)
- `template?.header_email` (fallback: `company?.email`)
- `template?.header_licence_no` (fallback: `company?.licence_no`)
- `template?.header_ntn` (fallback: `company?.ntn`)
- `template?.document_title` (fallback: 'TAX INVOICE')

### 3.2 Footer Template

**File**: `/mnt/c/Codes/Powersuite/psback/views/pages/invoiceFooter.ejs`

```ejs
<div class="footer">
  <div class="footer-content">
    <% if (template?.footer_text) { %>
      <div style="white-space: pre-line;"><%= template.footer_text %></div>
    <% } else { %>
      <div><%= company?.name || '' %></div>
      <div><%= company?.address || '' %> | Phone: <%= company?.phone || '' %></div>
    <% } %>
  </div>
</div>
```

**Key Feature**: 
- `white-space: pre-line` preserves line breaks in footer text
- Supports multi-line footers

### 3.3 Invoice Document Template

**File**: `/mnt/c/Codes/Powersuite/psback/views/pages/invoiceDocument.ejs` (48KB, 1000+ lines)

**Header Usage** (Lines 289-296):
```ejs
<div class="print-header">
  <div class="header--logo-text">
    <%= template?.header_name || invoices[0]?.Service?.Order?.user?.company?.name %>
  </div>
  <div class="logo--address">
    <%= template?.header_address || invoices[0]?.Service?.Order?.user?.company?.address %>
    <br>
    Phone: <%= template?.header_phone || invoices[0]?.Service?.Order?.user?.company?.phone %>
  </div>
</div>
```

**Footer Usage**: Similar pattern with `<div class="print-footer">`

**Special Features**:
- Multi-page support with fixed headers/footers
- "VOID" watermark for voided invoices
- CSS media queries for print formatting
- Proper margin management for multi-page documents

---

## 4. PDF Generation

**File**: `/mnt/c/Codes/Powersuite/psback/services/pdf.js`

**PDF Creation Flow**:

```javascript
exports.createPdf = async (htmlContent, landscape = false, customOptions = {}) => {
  // Uses wkhtmltopdf library
  // Supports custom margins, header/footer HTML
  // Returns PDF buffer
  
  const options = {
    pageSize: 'A4',
    orientation: landscape ? 'Landscape' : 'Portrait',
    marginTop: customOptions.marginTop || '10mm',
    marginRight: customOptions.marginRight || '10mm',
    marginBottom: customOptions.marginBottom || '10mm',
    marginLeft: customOptions.marginLeft || '10mm',
    enableLocalFileAccess: true,
    printMediaType: true,
    disableSmartShrinking: true,
    headerFontSize: 8,
    footerFontSize: 8,
    // Support for headerHtml and footerHtml
  };
};
```

**Header/Footer in PDF Generation**:

```javascript
exports.generateDocumentPDF = async (documentNumber) => {
  // Fetch HTML content from /document/:documentNumber
  // Fetch header HTML from /document/:documentNumber/header
  // Fetch footer HTML from /document/:documentNumber/footer
  
  const pdfOptions = {
    marginTop: headerHtml ? '35mm' : '10mm',
    marginBottom: footerHtml ? '25mm' : '10mm',
    ...(headerHtml && { headerHtml, headerSpacing: 0 }),
    ...(footerHtml && { footerHtml, footerSpacing: 0 })
  };
};
```

---

## 5. Frontend Implementation

### 5.1 API Client

**File**: `/mnt/c/Codes/Powersuite/psfront/src/api/branch_document_template.js`

```javascript
// Endpoints
getTemplatesByBranch(branchId)
getTemplateByBranchAndType(branchId, documentType)
updateTemplate(id, data)
updateTemplateByBranchAndType(branchId, documentType, data)
initializeTemplatesForBranch(branchId)
```

### 5.2 Templates List Component

**File**: `/mnt/c/Codes/Powersuite/psfront/src/pages/BranchDocumentTemplates/BranchDocumentTemplates.jsx`

**Features**:
- List all branches with accordion UI
- Search branches by name
- Lazy load templates on accordion expand
- "Initialize Templates" button for branches without templates
- Edit button for each template
- Displays: document type, title, header name

### 5.3 Template Editor Component

**File**: `/mnt/c/Codes/Powersuite/psfront/src/pages/BranchDocumentTemplates/EditBranchDocumentTemplate.jsx`

**Form Fields**:
```
[Required Fields]
- document_title (Input)
- header_name (Input)

[Header Section]
- header_phone (Input)
- header_email (Input)
- header_licence_no (Input)
- header_ntn (Input)
- header_address (Textarea - 3 rows)

[Footer Section]
- footer_text (Textarea - 4 rows)
```

**Features**:
- Branch and document type info displayed
- Form validation (title and name required)
- Toast notifications for success/error
- Navigation back to templates list

**Current Text Input**:
- Plain `<Input>` and `<Textarea>` components from Radix UI
- No rich text editor
- Supports multi-line input (pre-line formatting in footer)

---

## 6. Current Address and Footer Handling

### 6.1 Address Fields

**Storage**:
- `header_address` in `branch_document_template` (TEXT field, ~65K chars)
- Supports multi-line addresses
- Fallback to `company.address`

**Display in EJS**:
```ejs
<%= template?.header_address || company?.address || '' %>
```

**No Formatting**: Raw text display, no structured address components (no city/state/zip separate fields)

### 6.2 Footer Text

**Storage**:
- `footer_text` in `branch_document_template` (TEXT field)
- Supports any text content

**Display in EJS**:
```ejs
<div style="white-space: pre-line;"><%= template.footer_text %></div>
```

**Multi-line Support**: Yes, preserved via `white-space: pre-line` CSS

**Default Footer Generation** (in `branch_document_template.service.js`):
```javascript
footer_text: `${company.name || ''}\n${company.address || ''} | Phone: ${company.phone || ''}`
```

---

## 7. Current Text Editor Implementation

### 7.1 No WYSIWYG Editor Currently Used

**Plain Text Only**:
- EditBranchDocumentTemplate component uses simple `<Textarea>` component
- No formatting capabilities
- No inline styling

### 7.2 Available Libraries (Installed)

**Frontend** (`psfront/package.json`):
```json
"react-quill": "^2.0.0"  // Rich text editor - INSTALLED but NOT used for templates
```

**Usage**: Currently only used in `/psfront/src/components/AddTour.jsx` for tour descriptions

**Not Used Libraries** (Not installed):
- TinyMCE
- Draft.js
- Slate
- Prosemirror

---

## 8. Route Structure

### 8.1 Backend Routes

**File**: `/mnt/c/Codes/Powersuite/psback/routes/branch_document_template.route.js`

```javascript
GET    /branch-document-template/branch/:branchId
GET    /branch-document-template/branch/:branchId/type/:documentType
PUT    /branch-document-template/:id
PUT    /branch-document-template/branch/:branchId/type/:documentType
POST   /branch-document-template/branch/:branchId/initialize
```

**Auth**: All routes require JWT authentication

### 8.2 Document Rendering Routes

**File**: `/mnt/c/Codes/Powersuite/psback/routes/document.route.js`

```javascript
GET /:documentNumber/header    → DocumentController.getDocumentHeader
GET /:documentNumber/footer    → DocumentController.getDocumentFooter
```

---

## 9. Feature Summary Table

| Aspect | Current Implementation | Notes |
|--------|----------------------|-------|
| **Template Storage** | Database (branch_document_template) | Per-branch, per-document-type |
| **Header Fields** | 7 fields (name, address, phone, email, licence, ntn, title) | Text-based, no structured data |
| **Footer Support** | Single text field | Multi-line via pre-line CSS |
| **Address Format** | Flat text field | No separate city/state/zip |
| **Text Editor** | Plain textarea | No WYSIWYG |
| **Rich Text** | None | No formatting options |
| **Template Variables** | Fallback to company data | Limited to hard-coded fields |
| **PDF Headers/Footers** | EJS rendered HTML | Multi-page support |
| **Multi-line Support** | Yes (footer) | Via white-space: pre-line |
| **Validation** | Basic (required fields) | title and name required |
| **Versioning** | No | Only current version stored |
| **Preview** | No | No live preview before save |

---

## 10. File Locations Summary

### Backend Files

| File | Purpose |
|------|---------|
| `psback/models/system_models/branch_document_template.js` | Data model definition |
| `psback/services/branch_document_template.service.js` | Business logic |
| `psback/controllers/branch_document_template.controller.js` | API endpoints |
| `psback/routes/branch_document_template.route.js` | Route definitions |
| `psback/controllers/document.controller.js` | Document rendering (header/footer retrieval) |
| `psback/services/pdf.js` | PDF generation with wkhtmltopdf |
| `psback/views/pages/invoiceHeader.ejs` | Header template |
| `psback/views/pages/invoiceFooter.ejs` | Footer template |
| `psback/views/pages/invoiceDocument.ejs` | Main invoice document |
| `psback/models/invoice_footer.js` | Legacy footer model (not in use) |

### Frontend Files

| File | Purpose |
|------|---------|
| `psfront/src/api/branch_document_template.js` | API client |
| `psfront/src/pages/BranchDocumentTemplates/BranchDocumentTemplates.jsx` | Templates list page |
| `psfront/src/pages/BranchDocumentTemplates/EditBranchDocumentTemplate.jsx` | Template editor |

---

## 11. Database Query Examples

### Get Template
```sql
SELECT * FROM branch_document_template 
WHERE branch_id = ? AND document_type = ?
```

### Update Template
```sql
UPDATE branch_document_template 
SET document_title = ?, header_name = ?, header_address = ?, 
    header_phone = ?, header_email = ?, header_licence_no = ?, 
    header_ntn = ?, footer_text = ?, updated_at = NOW()
WHERE id = ?
```

### Get All Templates for Branch
```sql
SELECT * FROM branch_document_template 
WHERE branch_id = ? 
ORDER BY document_type ASC
```

---

## 12. Key Observations

### Strengths
1. ✓ Clean separation of concerns (models, services, controllers)
2. ✓ Branch-level customization with company fallback
3. ✓ Support for 7 document types
4. ✓ EJS templating with smart fallback logic
5. ✓ Multi-page PDF support with headers/footers
6. ✓ Proper constraint management (unique per branch/type)

### Limitations
1. ✗ No rich text editor for templates
2. ✗ Address is flat text field (no structured components)
3. ✗ No template versioning/history
4. ✗ No live preview before save
5. ✗ No template variables/placeholders (static content only)
6. ✗ No HTML editing capability
7. ✗ No image/logo support in headers/footers
8. ✗ Footer text is plain text only

### Recommendations for Enhancement
1. Add React Quill editor for rich text (already installed)
2. Implement structured address fields
3. Add template preview functionality
4. Support dynamic template variables ({{company_name}}, {{date}}, etc.)
5. Add image upload for logos/company branding
6. Implement template versioning
7. Add pre-defined templates/themes
8. Support HTML editing with validation

---

## 13. Dependencies

### Backend
- `ejs`: Template rendering
- `wkhtmltopdf`: PDF generation
- `sequelize`: ORM
- `mysql2`: Database driver

### Frontend
- `react-quill`: Rich text editor (installed but not used for templates)
- `react-hook-form`: Form handling
- `zod`: Validation
- `sonner`: Toast notifications
- `axios`: HTTP client

---

## Technical Flow Diagram

```
[Edit Template Page] 
    ↓
[React Hook Form + Textarea]
    ↓
[updateTemplateByBranchAndType API]
    ↓
[Express Route: PUT /branch/:branchId/type/:documentType]
    ↓
[Controller: updateTemplateByBranchAndType]
    ↓
[Service: updateTemplateByBranchAndType]
    ↓
[Sequelize Model: Update]
    ↓
[MySQL: branch_document_template table]

---

[PDF Generation]
    ↓
[Request: GET /document/:documentNumber]
    ↓
[Controller: getInvoiceDocument]
    ↓
[Service: getTemplateByBranchAndType]
    ↓
[EJS Render: invoiceDocument.ejs with template data]
    ↓
[wkhtmltopdf: HTML → PDF]
    ↓
[MinIO/S3: Store PDF]
    ↓
[PDF URL returned to client]
```

