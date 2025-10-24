# n8n Workflow: DrChrono Discharge Sheet Generator

This n8n workflow automates the generation of a patient discharge instruction sheet by fetching, reconciling, and merging data from a DrChrono electronic health record (EHR) system.

It is triggered by a webhook, fetches data from multiple DrChrono API endpoints (patient, appointment, clinical note, checkout sheet), and intelligently reconciles them into a single, comprehensive PDF document. The final document is then uploaded back to the patient's chart in DrChrono, saved to Google Cloud Storage, and a notification is sent to Slack.

## Key Features

* **Webhook Trigger:** Initiated by a `POST` request, allowing integration with DrChrono webhooks or other systems.
* **Status Validation:** Checks if the appointment status is `REFERRALS COMPLETED` before processing.
* **Data Reconciliation:** Intelligently merges data from the **Clinical Note** and the **Checkout Sheet**. It compares fields like diagnoses and procedures, flags conflicts, and uses authoritative sources for each section.
* **PDF Generation:** Creates a clean, styled HTML discharge sheet and converts it to a PDF.
* **Multi-Destination Upload:**
    * Uploads the final PDF to the patient's document section in DrChrono.
    * Backs up the PDF to a Google Cloud Storage bucket.
* **Auditing & Notification:**
    * Sends a Slack message to a specified channel with a summary of the generated sheet, including any data conflicts.
    * Generates a JSON audit log of the transaction.
* **Webhook Response:** Responds to the initial webhook caller with a success message, a link to the GCS document, and a summary of any conflicts.

## Workflow Overview

1.  **Trigger:** A webhook receives a `POST` request with `appointment_id`, `patient` (ID), and `appointment_status`.
2.  **Validate:** An `IF` node checks if `appointment_status` is `REFERRALS COMPLETED`. If not, it returns a 400 error.
3.  **Authenticate:** Gets a DrChrono OAuth2 access token using `client_credentials`.
4.  **Fetch Data:** Makes four parallel API calls to DrChrono:
    * Fetch Appointment Data
    * Fetch Patient Data
    * Fetch Clinical Note
    * Fetch Checkout Sheet
5.  **Reconcile:** A `Code` node merges the clinical note and checkout sheet. It identifies conflicts, calculates an "accuracy" percentage, and builds a single `reconciled` JSON object.
6.  **Generate HTML:** A `Code` node injects the `reconciled` data into a styled HTML template.
7.  **Convert to PDF:** The HTML is converted into an A4-formatted PDF.
8.  **Distribute & Save:** The PDF is uploaded to both Google Cloud Storage and the patient's chart in DrChrono.
9.  **Audit & Notify:** An audit log is created, and a Slack notification is sent.
10. **Respond:** A 200 OK JSON response is sent back to the webhook caller.

## Prerequisites

To use this workflow, you will need:

* An active n8n instance (self-hosted or cloud).
* **DrChrono API Credentials:** A `Client ID` and `Client Secret` with the necessary scopes to read patient/appointment data and write documents.
* **Google Cloud Storage:** A GCS bucket and a service account with permissions to write objects to it.
* **Slack:** A Slack workspace and credentials (Bot Token) to post messages to a channel.

## Installation

1.  Download the `n8n_discharge_workflow.json` file from this repository.
2.  In your n8n canvas, click on `Import` > `Import from File`.
3.  Select the downloaded JSON file. The workflow will be imported into your canvas.

## Configuration

You must configure the credentials for the nodes to work.

### 1. DrChrono Authentication

The node `Get DrChrono OAuth Token` is hardcoded with `client_id` and `client_secret`. **This is not secure for production.**

It is **highly recommended** to replace these with n8n's built-in credential management:

1.  In n8n, go to **Credentials** > **New**.
2.  Search for `Generic Credential Type` and select `Header Auth`.
3.  Name it something like "DrChrono Client/Secret".
4.  Add two "Name-Value Pair" fields:
    * `Name`: `clientId`, `Value`: [Your DrChrono Client ID]
    * `Name`: `clientSecret`, `Value`: [Your DrChrono Client Secret]
5.  Save the credential.
6.  In the `Get DrChrono OAuth Token` node, change the `client_id` and `client_secret` values to use expressions:
    * `client_id`: `={{$credentials.MyDrChronoCreds.clientId}}`
    * `client_secret`: `={{$credentials.MyDrChronoCreds.clientSecret}}`
    *(Replace `MyDrChronoCreds` with the name you gave your credential).*

### 2. Google Cloud Storage

1.  In n8n, go to **Credentials** > **New**.
2.  Search for and select **Google Cloud Storage**.
3.  Create a credential using your GCS service account JSON.
4.  In the n8n workflow, select the `Save to Google Cloud Storage` node.
5.  Choose your newly created GCS credential from the **Credential** dropdown.
6.  Set the **Bucket Name** parameter to your desired bucket.

### 3. Slack

1.  In n8n, go to **Credentials** > **New**.
2.  Search for and select **Slack**.
3.  Create a credential using your Slack Bot Token.
4.  In the n8n workflow, select the `Send Slack Notification` node.
5.  Choose your Slack credential from the **Credential** dropdown.
6.  Update the `Channel` parameter to the name of your Slack channel (e.g., `#discharge-notifications`).

## How to Use

Activate the workflow in n8n. The workflow is now listening for requests.

To trigger it, send a `POST` request to its webhook URL. You can find the URL in the `Webhook Trigger` node (e.g., `https://your-n8n.instance.com/webhook/generate-discharge`).

The `POST` request must have a JSON body containing the following fields:

```json
{
  "appointment_id": 123456,
  "patient": 789012,
  "appointment_status": "REFERRALS COMPLETED"
}
```

* `appointment_id`: The ID of the DrChrono appointment.
* `patient`: The ID of the DrChrono patient.
* `appointment_status`: The status of the appointment. The workflow will **only** run if this is set to `REFERRALS COMPLETED`.

### Success Response

If successful, the workflow will respond with a `200 OK` and a JSON body like this:

```json
{
  "success": true,
  "message": "Discharge sheet generated successfully",
  "patient": "Jane Doe",
  "appointment_id": 123456,
  "document_url": "[https://storage.googleapis.com/](https://storage.googleapis.com/)...",
  "accuracy": "83.33%",
  "conflicts": 1,
  "timestamp": "2025-10-25T01:30:00.000Z"
}
```

### Error Response

If the `appointment_status` is not correct, it will respond with a `400 Bad Request`:

```json
{
  "success": false,
  "message": "Appointment status must be 'REFERRALS COMPLETED'",
  "current_status": "Checked In"
}
```
