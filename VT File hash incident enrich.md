
---

# SentinelOne HyperAutomation Workflow Guide
![image](https://github.com/user-attachments/assets/deb2780c-1ae0-4e2b-8988-2db2871b3ed9)

## Overview
This workflow performs automated threat intelligence actions:
- Responds to SentinelOne alerts triggered by file hashes.
- Extracts the file hash from the alert.
- Searches for threat intelligence data tied to the hash.
- Generates a verdict note based on the intelligence.
- Appends the note to the SentinelOne alert for easier tracking and response.

---

## Step-by-Step Guide

### Step 1: Create a New Workflow
1. Log in to **SentinelOne HyperAutomation**.
2. Navigate to the **Automation Studio** and click **Create Workflow**.
3. Name the workflow (e.g., `File Hash Intelligence Automation`) and provide a description if necessary.

---

### Step 2: Set the Trigger
1. Drag and drop the **Singularity Response Trigger** node onto the canvas.
2. Open the configuration panel and set the trigger to respond to appropriate event feeds (e.g., file hash appearance in the “process” data).

---

### Step 3: Define Hash as a Variable
1. Drag the **Set Variable** node onto the canvas.
2. Configure the variable:
   - **Name**: `hash`
   - **Value**:
     ```plaintext
     {{(function.getAPI('singularity-response', 'trigger-data.process.file_hash'))}}
     ```
![image](https://github.com/user-attachments/assets/a103ada4-55e0-4378-81eb-f583a819e0f6)

---

### Step 4: Add a Condition to Validate the Hash
1. Drag the **Condition** node to the canvas and connect it after the variable definition.
2. Set the condition to ensure the hash exists:
   - **Value A**: `{{local.var.hash}}`
   - **Condition**: **Not Equal**
   - **Value B**: `null`

#### If the hash is `null`:
- Add a message to the alert note.

#### If the hash exists (is not null):
- Proceed to the next steps.

---

### Step 5: Search the File Hash
1. Drag the **Search File Hash** node onto the canvas.
2. Configure the node to query a threat intelligence source (e.g., VirusTotal):
   - **API Endpoint**:
     ```
     https://www.virustotal.com/api/v3/files/{{local.var.hash}}
     ```
   - **Headers**:
     - **Key**: `Content-Type`
     - **Value**: `application/json`
3. **Note**: Instructions to set up SOC integrations can be found in the document located in the ` integrations folder`.

---

### Step 6: Generate the Verdict Note
1. Drag the **Set Variable** node onto the canvas.
2. Configure this node:
   - **Name**: `note`
   - **Value**:
     ```plaintext
     A VirusTotal lookup on the file hash {{local.var.hash}} provides the following details:
     Metadata: {{results.data.metadata}}
     Classification: {{results.data.classification}}
     First Seen: {{results.data.first_seen}}
     ```
![image](https://github.com/user-attachments/assets/320c9fa0-7b54-425b-bc7c-3ac9c0ddd2e6)

---

### Step 7: Encode the Verdict Note
1. Drag another **Set Variable** node onto the canvas.
2. Configure this variable:
   - **Name**: `encoded`
   - **Value**:
     ```plaintext
     {{function.WHEN_TO_ENCODE(local.var.note)}}
     ```

---

### Step 8: Add Verdict Note to SentinelOne Alert
1. Drag the **Add Verdict Note to Alert** node onto the canvas.
2. Configure the node:
   - **API Endpoint**:
     ```
     https://<your-instance>.sentinelone.net/web/api/v2.1/alerts/{alert_id}
     ```
     Replace `<your-instance>` with your SentinelOne instance URL.
   - **Headers**:
     - **Key**: `Content-Type`
     - **Value**: `application/json`
   - **Body**:
     ```json
     {
       "query": "mutation AddVerdictNote($alertId: String!, $note: String!) { addAlertNote( alertId: $alertId, note: $note ) }",
       "variables": {
         "alertId": "{{singularity-response.trigger.data.alertId}}",
         "note": "{{local.var.encoded}}"
       }
     }
     ```
![image](https://github.com/user-attachments/assets/cb15b41a-ef9a-40ce-9a30-8f89ffe342b2)

---

## Workflow Diagram
Here is an overview of the workflow structure:

```
Singularity Response Trigger
    ↓
Set Variable (hash)
    ↓
Condition (Validate hash is not null)
    ↓
Search File Hash
    ↓
Set Variable (note)
    ↓
Encode Note
    ↓
Add Verdict Note to Alert
```

---
