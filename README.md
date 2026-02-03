Lead Generation and VIP Filtering System

This n8n workflow automates the ingestion, cleaning, and qualification of inbound leads via Webhook (e.g., from Postman). It filters leads based on budget and industry criteria, automatically segmenting high-value clients into a VIP database.

  Key Features
- Webhook Ingestion: Receives bulk or single lead data via HTTP POST requests.
- Data Normalization: Uses "Split Out" to handle arrays and "Set" to standardize data fields.
- Multi-Stage Filtering:
  1. Removes invalid entries via a preliminary Filter node.
  2. Blocks competitors using a conditional logic (If node).
- VIP Qualification: Routes leads based on specific criteria (Budget > Threshold AND IT Sector).
- Database Integration: Automatically saves qualified VIP leads into a Google Sheet.

 Workflow Logic
1. Webhook: Triggers the workflow with incoming JSON data.
2. Split Out: Separates batched data into individual items.
3. Data Cleaning (Set): Formats fields to ensure consistency.
4. Filter: Basic validation to drop incomplete records.
5. Competitor Check (If): Excludes leads from competitor domains.
6. VIP Verification (If): Checks if the lead meets high-budget and IT sector criteria.
7. Storage: Adds the qualified VIP lead to the Google Sheet.

 Nodes Used
- Webhook
- Split Out
- Set (Data Cleaning)
- Filter
- If (Competitor Check & VIP Verification)
- Google Sheets

Built with n8n by Fabio Roggero
