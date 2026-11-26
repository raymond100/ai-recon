# Freight Invoice Field Research

Status: Domain research for planning; proposed pack decisions require maintainer review  
Last updated: 2026-11-26

## Purpose and boundary

This note evaluates the sample freight invoice fields against X12 210 semantics and carrier implementation guides. X12 210 is a transaction structure for motor-carrier freight details and invoice; it is not a universal PDF invoice schema. Partner guides can impose different required usage. The pack should therefore describe normalized business concepts and retain the source/carrier values and evidence.

## Findings

1. **Invoice number is a strong required field.** X12 210's B3 Beginning Segment is mandatory and B302 Invoice Number is mandatory in the cited Kroger implementation guide. Some guides describe an invoice or carrier reference as a PRO; therefore preserve the document's invoice number and PRO/carrier reference as separate concepts rather than assuming they are interchangeable. [Kroger X12 210 guide](https://edi.kroger.com/EDIPortal/documents/Maps/ht/HT_210_X12_4020_Guidelines.pdf) [One Network X12 210 guide](https://partnerservices.onenetwork.com/wp-content/uploads/2024/05/X12_210_4010_Motor-Carrier-Freight-Details-and-Invoice-v1.0.pdf)

2. **BOL, shipment ID, and PRO are distinct references.** X12's 210 B3 has a Shipment Identification Number element, while N9 reference qualifiers include BM (Bill of Lading Number) and CN (Carrier's Reference Number / PRO or invoice in the cited partner guide). Keep `bol_number`, `shipment_id`, `pro_number`, and `invoice_number` as distinct nullable fields with reference type, issuer/source, and evidence. Do not collapse BOL into a generic shipment ID or treat PRO as a synonym for invoice number. [SAP X12 210 schema guide](https://help.sap.com/doc/message-implementation-guideline-ansi-x12-210/LBN/en-US/SAP_LBN_ANSI_X12_210.pdf) [One Network X12 210 guide](https://partnerservices.onenetwork.com/wp-content/uploads/2024/05/X12_210_4010_Motor-Carrier-Freight-Details-and-Invoice-v1.0.pdf)

3. **Carrier SCAC is a useful carrier identifier, but availability should be conditional.** X12 210 supports the Standard Carrier Alpha Code in B311; implementation usage varies. NMFTA describes SCAC as a two-to-four letter carrier identifier and the organization responsible for issuing/managing it. Keep normalized SCAC uppercase, preserve the raw value, and allow other carrier IDs/names when SCAC is absent. [SAP X12 210 schema guide](https://help.sap.com/doc/message-implementation-guideline-ansi-x12-210/LBN/en-US/SAP_LBN_ANSI_X12_210.pdf) [NMFTA SCAC](https://scac.nmfta.org/about)

4. **Use “shipment method of payment,” not generic payment method.** In X12, element 146 identifies terms for transportation charges. PP means prepaid by seller, CC collect, and TP third-party pay. Other code values exist, so accept the standard code set and map common document wording to a code only when supported by evidence; don't imply these are the only valid values. [DoD transportation EDI convention](https://www.ustranscom.mil/cmd/associated/dteb/files/transportationics/dteb858d004010_Version_8.pdf) [SAP X12 210 schema guide](https://help.sap.com/doc/message-implementation-guideline-ansi-x12-210/LBN/en-US/SAP_LBN_ANSI_X12_210.pdf)

5. **Model charges as repeated, extensible line items.** X12 210 carries repeated charge detail; FedEx's carrier guide includes L1 Rate and Charges. Use a general charge array with source code and description, normalized category, amount/currency, and any quantity, unit, rate, tax, or discount attributes present. Linehaul, fuel surcharge, and accessorial categories are useful normalized classifications, but preserve each original item and allow unknown/other codes. Avoid requiring both linehaul and fuel surcharge when a carrier invoice legitimately has a different structure. [FedEx X12 210 implementation guide](https://www.fedex.com/content/dam/fedex/us-united-states/shipping/images/EDI_X12_4060_210_Parcel-Motor_Carrier_Freight_Details_and_Invoice_and_820_Payment_Order_Remittance_Advice_Implementation_Guide.pdf)

6. **Dates, weights, amounts, and currency need semantic precision.** Keep invoice date distinct from ship/pickup/delivery dates. Store normalized dates as ISO dates when the source gives a date; if a timestamp or interval appears, preserve its time/precision and qualifier. Store money as Decimal with explicit currency and original value; do not irreversibly round extracted values to two decimal places before comparison. Apply currency-specific rounding/tolerance in deterministic rules. Store weight as Decimal plus unit, preserve source unit, and normalize recognized units (including LB/KG) without losing provenance.

7. **Parties and route should be role-based structured values.** Represent shipper, consignee, and bill-to as separate party roles, with available address and identifier components; represent origin/destination as locations. City/state alone may be insufficient for matching. Keep freight class, pieces, trailer/equipment identifiers, load/PO references, and comments optional or pack/customer-specific.

## Proposed extraction profile

Use tiering as a product-pack policy, not as a claim that X12 or every carrier requires the field.

### Core identity and reconciliation fields
- `invoice_number`: required; source-assigned invoice identifier.
- `invoice_date`: conditionally required where present in the source/tenant workflow; separate from shipment dates.
- `total_amount` and `currency`: required for financial reconciliation; preserve Decimal precision and source currency.
- `carrier`: require a resolvable carrier identity for carrier-specific rate checks; prefer SCAC when available, but permit another tenant-known identifier plus legal/display name.
- `shipment_references[]`: extract typed references (BOL, PRO/carrier reference, shipment ID, load, PO); require the particular key only when the tenant's configured matching rule needs it.

### Common matching and explanation fields
- Shipment method of payment (X12 element 146), pickup/ship/delivery dates with qualifiers, shipper/consignee/bill-to parties, origin/destination locations, total weight and unit, pieces, service level, and freight class when shown.
- Charge line array with source code/description, normalized category, amount, currency, and optional quantity/unit/rate/tax/discount.
- Trailer/equipment number and free-text comments only when available and useful to the configured workflow.

### Normalization and evidence
- Preserve both raw extracted text/value and normalized value. Each value keeps confidence, source quote/evidence locator, and source page/region when the parser provides it.
- Trim surrounding whitespace for identifiers; uppercase SCAC and currency codes; retain leading zeroes and meaningful internal punctuation/spacing.
- Normalize code values only when confidently identified. Payment code is shipment method of payment (146), not tender/payment rail.
- Parse monetary quantities as Decimal and retain source precision; never stringify binary floating-point values or silently force every value to two decimal places.
- Normalize dates to ISO 8601 representations while preserving date/time precision and the source qualifier.
- Pair each quantity with its unit. Keep original unit/value alongside any normalized form.
- Preserve unknown charge codes, labels, parties, and reference qualifiers so future pack versions can map them without re-extracting documents.

## Open design decisions

- Which representative carrier invoice PDFs define the initial golden corpus, including parcel, LTL, and truckload scope?
- Is SCAC required only for carrier-rate comparison, or must every MVP invoice resolve to a tenant carrier master record?
- Which matching references will the pilot master data provide (PO, BOL, PRO, load, shipment ID), and which are mandatory by tenant/rule?
- Which charge taxonomy and tax/discount/credit treatment are needed for the pilot?
- What currency rounding and reconciliation tolerance should apply per currency and rule?
- Do we include parcel invoices in the first freight pack, or limit to motor-carrier LTL/TL invoices?

## Sources and limits

The cited sources include X12 210 schema documentation and carrier/trading-partner implementation guides. Implementation guides show partner usage, not a universal requirement for all freight invoices. This research does not replace licensed/current ASC X12 code-list materials or review of the target carriers' actual invoice samples.
