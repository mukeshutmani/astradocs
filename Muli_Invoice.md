Multiple Invoicing for a Service - Summary

  Overview

  The system supports multiple invoices per service, allowing for invoice voiding and regeneration while maintaining a complete audit trail.

  Database Structure

  - Relationship: One Service can have many Invoices (1:N relationship)
  - Association: Service.hasMany(Invoice) with foreign key service_id
  - Status Field: Each invoice has a status field that can be:
    - Raised - New invoice, not yet printed
    - Printed - Invoice has been printed/finalized
    - Void - Invoice has been voided/cancelled
    - Settled - Invoice has been paid
    - Partially Settled - Partially paid

  How It Works

  1. Initial Invoice Creation

  - When a service is created, an invoice is generated with status Raised
  - The invoice stores all pricing details including base price, markup, discounts, taxes, and calculates total_price

  2. Invoice Voiding Process

  When an invoice needs to be voided (e.g., due to errors or changes):
  - The original invoice's status is changed to Void
  - A new replacement invoice is automatically created with:
    - Same pricing details as the voided invoice
    - Recalculated total_price
    - Status set to Raised
    - New invoice number (auto-generated)

  3. Active Invoice Selection

  The system identifies the "active" invoice using:
  const activeInvoice = service?.Invoices
    ?.filter(inv => inv.status !== 'Void')  // Exclude voided invoices
    ?.sort((a, b) => {                      // Get the most recent
      if (a.createdAt && b.createdAt) {
        return new Date(b.createdAt) - new Date(a.createdAt);
      }
      return (b.id || 0) - (a.id || 0);
    })?.[0];

  4. Display Logic

  - Service Tables: Show the total_price from the active (non-voided) invoice
  - Invoice Report: Lists only invoices with Raised status for document generation
  - Service Details: Displays pricing information from the latest active invoice

  5. Document Generation

  - Only invoices with Raised status can be selected for document generation
  - Once printed, the invoice status changes to Printed
  - Printed invoices cannot be edited but can be voided if needed

  Key Benefits

  1. Audit Trail: All invoices are preserved, including voided ones
  2. Error Correction: Mistakes can be corrected by voiding and regenerating
  3. Compliance: Maintains complete history for accounting/legal requirements
  4. Flexibility: Allows multiple attempts at getting invoice details correct before printing

  Business Rules

  - Only one active (non-voided) invoice per service at a time
  - Cannot create a new invoice if an active one exists (must void first)
  - Voided invoices remain in the system for record-keeping
  - Invoice numbers are never reused
  - The total_price field stores the final calculated amount to avoid recalculation inconsistencies

  API Endpoints

  - GET /service/:id - Returns service with all invoices (Invoices array)
  - POST /invoice - Creates new invoice (blocked if active invoice exists)
  - PUT /invoice/void/:document_number - Voids invoice and creates replacement
  - GET /invoice/order/listAllServiceInvoices/:orderId - Lists invoices for document generation

  This architecture ensures data integrity while providing flexibility for corrections and maintaining a complete financial history.