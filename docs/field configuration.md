# Field Configuration Guide

This document provides detailed instructions for configuring all custom fields required by the Dynamics 365 Agentic Claims Orchestration workflow in your Dynamics 365 Customer Service environment.

---

## Overview

The workflow requires custom fields to be added to the `incident` (Case) table to store:
- Claim metadata (Claim Type, Claim Amount, Claim Summary)
- Agent analysis outputs (Fraud Score, Coverage Status, AI Recommendation, Customer Communication)
- Operational decisions (Status Reason values, Approval Status)

All fields are configured with the schema prefix `cr7ba_` to avoid conflicts with standard Dynamics 365 fields.

---

## Prerequisites

Before configuring fields, ensure you have:
- **System Administrator** or **System Customizer** role in the Dynamics 365 organization
- Access to the **Power Apps** maker portal (make.powerapps.com)
- The target Dynamics 365 Customer Service environment
- Understanding of field types and column configurations in Dataverse

---

## Table of Contents

1. [Claim Input Fields](#claim-input-fields)
2. [Agent Output Fields](#agent-output-fields)
3. [Decision & Classification Fields](#decision--classification-fields)
4. [Field Mappings & Dependencies](#field-mappings--dependencies)
5. [Validation Rules](#validation-rules)
6. [Bulk Configuration Script](#bulk-configuration-script)

---

## Claim Input Fields

These fields capture the insurance claim data that the agent will reason about. They are populated either at Case creation (via form, email-to-Case, omnichannel) or by upstream automation.

### 1. Claim Type

**Display Name:** Claim Type  
**Logical Name:** `cr7ba_claimtype`  
**Field Type:** Single Line of Text  
**Required:** Yes  
**Searchable:** Yes  
**Max Length:** 100 characters  
**Description:** The line of insurance or claim category (e.g., Vehicle Insurance, Home Insurance, Health Claim)

**Configuration Steps:**

1. Navigate to **Power Apps** → **Solutions** → **Default Solution** (or your custom solution)
2. Select **Tables** → **Case** (incident)
3. Click **+ New** → **Column**
4. Fill in:
   - **Display name:** Claim Type
   - **Name:** cr7ba_claimtype (auto-generated)
   - **Data type:** Text → Single line of text
   - **Required:** Business required
   - **Searchable:** Yes
5. Click **Save and Close**

**Form Integration:**
- Add this field to the Case form under a "Claim Details" section
- Make it visible and required on form load
- Optionally add a pre-defined choice list if your insurance lines are fixed

### 2. Claim Amount

**Display Name:** Claim Amount (Base)  
**Logical Name:** `cr7ba_claimamount`  
**Field Type:** Currency  
**Required:** Yes  
**Currency:** EUR (Euros)  
**Min Value:** 0  
**Max Value:** 9,999,999.99  
**Decimal Precision:** 2  
**Description:** The monetary amount being claimed, in euros

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Claim Amount (Base)
   - **Name:** cr7ba_claimamount (auto-generated)
   - **Data type:** Currency
   - **Currency:** Euro (EUR)
   - **Behavior:** Currency
   - **Precision:** 2 decimal places
   - **Required:** Business required
3. Under **Validation**, set minimum value to **0**
4. Click **Save and Close**

**Important Notes:**
- Dynamics 365 automatically creates a shadow field `cr7ba_claimamount_base` for exchange rate-adjusted values. The agent receives the base currency value.
- If your organization uses multi-currency, ensure the exchange rate is set correctly before Case creation.
- Display currency is EUR; conversion happens at the org level.

**Form Integration:**
- Add to the Claim Details section
- Format as currency with EUR symbol
- Make required and visible on form load

### 3. Claim Summary

**Display Name:** Claim Summary  
**Logical Name:** `cr7ba_claimsummary`  
**Field Type:** Multiple Lines of Text  
**Required:** Yes  
**Max Length:** 2,000 characters  
**Description:** Free-text description of the incident, what happened, and key details

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Claim Summary
   - **Name:** cr7ba_claimsummary (auto-generated)
   - **Data type:** Text → Multiple lines of text
   - **Format:** Text area
   - **Max length:** 2000
   - **Required:** Business required
3. Click **Save and Close**

**Form Integration:**
- Add to Claim Details section below Claim Amount
- Use a large text area (6-8 rows visible)
- Add a tooltip: "Describe what happened, when, where, and any other relevant details"

---

## Agent Output Fields

These fields store the agent's reasoning outputs. They are populated by the Power Automate flow after the agent runs. Handlers should treat these as read-only recommendations, not source-of-truth decisions.

### 1. Fraud Score

**Display Name:** Fraud Score  
**Logical Name:** `cr7ba_fraudscore`  
**Field Type:** Whole Number  
**Required:** No  
**Min Value:** 0  
**Max Value:** 100  
**Description:** AI-assigned fraud risk score (0 = no risk, 100 = certain fraud)

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Fraud Score
   - **Name:** cr7ba_fraudscore (auto-generated)
   - **Data type:** Whole Number
   - **Language:** Not applicable
   - **Behavior:** None
   - **Min value:** 0
   - **Max value:** 100
   - **Required:** No
3. Click **Save and Close**

**Form Integration:**
- Add to an "AI Analysis" section (read-only)
- Display with a visual indicator (e.g., red for >70, amber for 25-70, green for <25)
- Use conditional formatting if the form supports it
- Accompany with a tooltip: "0-100 scale assigned by AI agent. Higher scores indicate higher fraud risk."

**Calibration Reference:**
- **0-15:** Routine claim, no fraud indicators
- **16-25:** Minor ambiguity, no concerning patterns
- **26-50:** One or more soft indicators warranting review
- **51-75:** Multiple indicators or one strong indicator
- **76-100:** Clear signs of likely fraud

### 2. Coverage Status

**Display Name:** Coverage Status  
**Logical Name:** `cr7ba_coveragestatus`  
**Field Type:** Single Line of Text  
**Required:** No  
**Max Length:** 50 characters  
**Description:** AI assessment of whether the claim is likely covered under the policy

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Coverage Status
   - **Name:** cr7ba_coveragestatus (auto-generated)
   - **Data type:** Text → Single line of text
   - **Max length:** 50
   - **Required:** No
3. Click **Save and Close**

**Allowed Values (Enumerated):**

The agent returns one of exactly three values:

| Value | Meaning |
|---|---|
| `Likely Covered` | Claim type and circumstances match typical coverage patterns |
| `Likely Not Covered` | Claim shows clear indicators of typical exclusions |
| `Unclear` | Insufficient information or ambiguous situation |

**Form Integration:**
- Add to AI Analysis section (read-only)
- Consider adding a choice column (next item) for a more formal enumeration if you want dropdown enforcement
- Display with color coding: green for Likely Covered, red for Likely Not Covered, amber for Unclear

**Alternative Implementation:**
Consider creating a Choice column instead if you want stricter schema:
- Field Type: Choice
- Options: Likely Covered | Likely Not Covered | Unclear
- This prevents invalid values but requires schema change in the agent's output schema

### 3. AI Recommendation

**Display Name:** AI Recommendation  
**Logical Name:** `cr7ba_airecommendation`  
**Field Type:** Multiple Lines of Text  
**Required:** No  
**Max Length:** 1,000 characters  
**Description:** The agent's natural-language reasoning and next operational step

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** AI Recommendation
   - **Name:** cr7ba_airecommendation (auto-generated)
   - **Data type:** Text → Multiple lines of text
   - **Format:** Text area
   - **Max length:** 1000
   - **Required:** No
3. Click **Save and Close**

**Form Integration:**
- Add to AI Analysis section (read-only)
- Display as a large read-only text area (4-5 rows)
- This is the primary explainability field — handlers rely on this to understand why the agent made its decision

**Example Outputs:**
- "Claim consistent with standard motor collision coverage; verify active policy and standard exclusions during adjuster review."
- "Hold payment and route to manual review due to anomalous claim amount and potential fraud indicators before any payout approval."
- "Coverage status unclear; request police report and incident timeline before processing."

### 4. Customer Communication

**Display Name:** Customer Communication  
**Logical Name:** `cr7ba_customercommunication`  
**Field Type:** Multiple Lines of Text  
**Required:** No  
**Max Length:** 2,000 characters  
**Description:** Draft customer-facing message for handler review and potential sending

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Customer Communication
   - **Name:** cr7ba_customercommunication (auto-generated)
   - **Data type:** Text → Multiple lines of text
   - **Format:** Text area
   - **Max length:** 2000
   - **Required:** No
3. Click **Save and Close**

**Form Integration:**
- Add to a "Communication" section below AI Analysis
- Display as read-only for initial review
- Add a button or link to copy the text to the Case description or timeline for sending
- Make it clear to handlers: "This is a draft. Review and edit before sending to customer."

**Example Outputs:**
- "Thank you for submitting your claim following the rear-end collision. We are carefully reviewing the details and documentation to ensure the right outcome, and our team may be in touch shortly if any additional information is needed."
- "We have received your claim regarding the recent vehicle incident and have initiated the review process. Our team is currently verifying the details and supporting documents."

---

## Decision & Classification Fields

These fields capture the operational decisions made by the Decision Engine (Power Automate Switch action). They drive routing, queue assignment, SLA timers, and downstream automation.

### 1. Approval Status

**Display Name:** Approval Status  
**Logical Name:** `cr7ba_approvalstatus`  
**Field Type:** Choice  
**Required:** No  
**Description:** AI-recommended approval status guiding handler decision

**Configuration Steps:**

1. Navigate to **Power Apps** → **Case table** → **+ New Column**
2. Fill in:
   - **Display name:** Approval Status
   - **Name:** cr7ba_approvalstatus (auto-generated)
   - **Data type:** Choice
   - **Required:** No
3. Under **Options**, add the following choices:
   - **Label:** Approval Required | **Value:** 383300001
   - **Label:** Approval Not Required | **Value:** 383300002
   - **Label:** Escalation Recommended | **Value:** 383300003
4. Set **Default choice:** (leave blank, allow null)
5. Click **Save and Close**

**Form Integration:**
- Add to Decision section (read-only when populated by flow)
- Display as a choice dropdown
- Color-code visually: green for Not Required, amber for Required, red for Escalation
- Use conditional visibility to only show when populated

**Mapping to Flow:**
The Power Automate flow's Decision Engine Switch action writes one of these three values based on the agent's `approvalrequired` boolean:
- `approvalrequired = false` → Approval Status = 383300002
- `approvalrequired = true` AND fraud score < 70 → Approval Status = 383300001
- `approvalrequired = true` AND fraud score >= 70 → Approval Status = 383300003

### 2. Status Reason (Standard Field - Enhanced)

**Display Name:** Status Reason  
**Logical Name:** `statuscode`  
**Field Type:** Choice (built-in, requires schema modification)  
**Note:** This is a standard Dynamics 365 field. You are adding new choice values, not creating a new field.

**Standard Status Reason Values:**
The `statuscode` field has default values tied to the `statecode` (Status) parent. For Claims, the standard value structure is:

| statecode | Default statuscode | Display |
|---|---|---|
| 0 (Active) | 1 | In Progress |
| 1 (Resolved) | 5 | Problem Solved |
| 2 (Cancelled) | 6 | Cancelled |

**Adding AI-Specific Status Reason Values:**

To add the three AI classification outcomes, you must add new choice values to the statuscode field under the Active (0) state.

**Configuration Steps:**

1. Navigate to **Power Apps** → **Solutions** → **Default Solution** (or custom solution)
2. Select **Tables** → **Case** (incident)
3. Select the **statuscode** column (double-click to edit)
4. Under **Choices**, scroll to the **Active (statecode: 0)** section
5. Click **+ Add** to add new choice values:

   **Choice 1:**
   - **Label:** AI Reviewed - Auto Approved
   - **Value:** 100000001
   - **External Value:** (leave blank)
   - **Color:** Green

   **Choice 2:**
   - **Label:** AI Reviewed - Pending Approval
   - **Value:** 100000002
   - **External Value:** (leave blank)
   - **Color:** Amber/Orange

   **Choice 3:**
   - **Label:** AI Reviewed - Needs Human Review
   - **Value:** 100000003
   - **External Value:** (leave blank)
   - **Color:** Red

6. Click **Save and Close**

**Important Notes:**
- The values `100000001`, `100000002`, `100000003` are examples. Use your organization's actual values if you've pre-allocated ranges for custom statuses.
- The parent `statecode` (Status) remains Active (0) — the AI never changes lifecycle status.
- These status values are **enum choices**, not free-text. The Power Automate flow writes the integer value, and Dynamics 365 displays the label.

**Form Integration:**
- The Status Reason field is usually already on the Case form (standard field)
- No additional configuration needed; the new values appear automatically in the dropdown
- Consider conditional formatting: hide this field until the AI has run, or mark it as "AI-Assigned" visually

**Downstream Automation:**
Create Business Process Flows or Queue Rules that react to these status values:
- Status Reason = 100000001 → Route to "Fast Track Approval" queue
- Status Reason = 100000002 → Route to "Standard Approval" queue
- Status Reason = 100000003 → Route to "Senior Review" queue

---

## Field Mappings & Dependencies

This section clarifies how fields flow through the system and depend on each other.

### Data Flow Map

```
┌─────────────────────────────────────────┐
│  Case Created (Input Fields)            │
│  • Claim Type                           │
│  • Claim Amount                         │
│  • Claim Summary                        │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Power Automate Trigger                 │
│  "When a row is added"                  │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Run an Agent (Insurance Operations)    │
│  Input: Claim Type, Amount, Summary     │
│  Output: Fraud Score, Coverage Status,  │
│          AI Recommendation,             │
│          Customer Communication,        │
│          Approval Required (boolean)    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Decision Engine (Switch on             │
│  approvalRequired boolean)              │
│  Output: Integer value for Status       │
│          Reason (100000001, 100000002,  │
│          100000003)                     │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Update a Row (Case)                    │
│  Write:                                 │
│  • cr7ba_fraudscore                     │
│  • cr7ba_coveragestatus                 │
│  • cr7ba_airecommendation               │
│  • cr7ba_customercommunication          │
│  • cr7ba_approvalstatus                 │
│  • statuscode                           │
└─────────────────────────────────────────┘
```

### Field Dependencies

| Field | Populated By | Triggers | Read-Only | Updated | Visible |
|---|---|---|---|---|---|
| Claim Type | Case creator or email-to-Case | Agent reasoning | No | No | Yes |
| Claim Amount | Case creator or email-to-Case | Agent reasoning | No | No | Yes |
| Claim Summary | Case creator or email-to-Case | Agent reasoning | No | No | Yes |
| Fraud Score | Power Automate (agent output) | Decision Engine | Yes | On agent re-run | Yes |
| Coverage Status | Power Automate (agent output) | Decision Engine | Yes | On agent re-run | Yes |
| AI Recommendation | Power Automate (agent output) | Handler follow-up | Yes | On agent re-run | Yes |
| Customer Communication | Power Automate (agent output) | Handler sends/edits | No | On agent re-run | Yes |
| Approval Status | Power Automate (Decision Engine) | Handler judgment | Yes | On flow re-run | Yes |
| Status Reason | Power Automate (Decision Engine) | Queue routing, SLA | Yes | Handler can override | Yes |

---

## Validation Rules

Validation rules enforce data quality and prevent invalid states. Configure these using Dataverse Business Rules or column validation.

### Column-Level Validation

**Claim Amount Validation:**
- Minimum: 0
- Maximum: 9,999,999.99
- Cannot be null
- Type: Currency

**Fraud Score Validation:**
- Minimum: 0
- Maximum: 100
- Type: Whole Number (Integer)
- Must be integer (no decimals)

**Coverage Status Validation:**
- Allowed values: "Likely Covered", "Likely Not Covered", "Unclear"
- Consider enforcing via Choice column instead of Text

### Business Rule: Approve Only Clean Claims

**Rule Name:** `cr7ba_AutoApprovalCriteria`

**Condition:**
When all of the following are true:
- Fraud Score <= 25
- Coverage Status = "Likely Covered"
- Claim Amount < 5000
- Status Reason is not already set

**Action:**
- Set Status Reason = 100000001 (Auto Approved)

**Note:** This rule prevents handlers from overwriting AI decisions after creation. Consider making it advisory (show a notification) rather than prescriptive (block the action).

### Business Rule: Escalate High Fraud

**Rule Name:** `cr7ba_EscalateHighFraud`

**Condition:**
When Fraud Score >= 70

**Action:**
- Set Approval Status = "Escalation Recommended"
- Change Status Reason = 100000003 (Needs Human Review)

---

## Bulk Configuration Script

If you are configuring multiple fields at once, you can use this XML to import via the **Solutions** interface, or use the Power Apps CLI to batch-create fields.

### Using Power Apps CLI

If you have the Power Apps CLI installed, you can use this script to create all fields:

```bash
# Authenticate
pac auth create -u <org-url>

# Create solution (if needed)
pac solution init -n "AgenAIC Claims" -p agenticclms

# Add fields using metadata API
# Note: This requires a solution.xml file; CLI is limited for bulk column creation
# Recommended: Use the UI for initial setup, then export as solution
```

### Manual Import via Solution XML

Create a `customizations.xml` with the following field definitions and import via **Solutions** → **Import**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<ImportExportXml>
  <Entities>
    <Entity>
      <Name>incident</Name>
      <Attributes>
        <!-- Claim Type Field -->
        <Attribute Name="cr7ba_claimtype" Type="String" MaxLength="100" Required="true" />
        
        <!-- Claim Amount Field -->
        <Attribute Name="cr7ba_claimamount" Type="Money" Required="true" MinValue="0" />
        
        <!-- Claim Summary Field -->
        <Attribute Name="cr7ba_claimsummary" Type="String" MaxLength="2000" Required="true" />
        
        <!-- Fraud Score Field -->
        <Attribute Name="cr7ba_fraudscore" Type="Integer" MinValue="0" MaxValue="100" />
        
        <!-- Coverage Status Field -->
        <Attribute Name="cr7ba_coveragestatus" Type="String" MaxLength="50" />
        
        <!-- AI Recommendation Field -->
        <Attribute Name="cr7ba_airecommendation" Type="String" MaxLength="1000" />
        
        <!-- Customer Communication Field -->
        <Attribute Name="cr7ba_customercommunication" Type="String" MaxLength="2000" />
        
        <!-- Approval Status Choice Field -->
        <Attribute Name="cr7ba_approvalstatus" Type="Picklist">
          <OptionSet>
            <Option Value="383300001">Approval Required</Option>
            <Option Value="383300002">Approval Not Required</Option>
            <Option Value="383300003">Escalation Recommended</Option>
          </OptionSet>
        </Attribute>
      </Attributes>
    </Entity>
  </Entities>
</ImportExportXml>
```

**Import Steps:**
1. Save the XML as `customizations.xml`
2. In **Power Apps**, go to **Solutions**
3. Click **Import** → select the XML file
4. Review and confirm the changes
5. Click **Import**

---

## Form Configuration

After fields are created, configure them on the Case form for optimal handler experience.

### Recommended Form Sections

**Section 1: Claim Details**
- Claim Type (required, visible, editable)
- Claim Amount (required, visible, editable, currency format)
- Claim Summary (required, visible, editable, large text area)

**Section 2: AI Analysis** (read-only)
- Fraud Score (read-only, with visual indicator 0-100)
- Coverage Status (read-only, color-coded)
- AI Recommendation (read-only, large text area)

**Section 3: Communication**
- Customer Communication (read-only initially, editable for handler to customize)

**Section 4: Decision** (read-only)
- Approval Status (read-only choice)
- Status Reason (read-only, auto-populated by flow)

### Form Business Rules

**Rule:** Hide AI Analysis section until agent has run
- Condition: `cr7ba_fraudscore` is null
- Action: Hide section

**Rule:** Show confirmation when Status Reason is set
- Condition: `statuscode` = 100000003 (Needs Human Review)
- Action: Show notification "This claim has been escalated. Please review AI Recommendation."

---

## Security & Permissions

Configure field-level security to prevent unauthorized access or modification.

### Field Security Roles

**Handler Role (Claims Handler):**
- Read: All fields
- Create: Claim Type, Claim Amount, Claim Summary
- Update: Customer Communication, Claim Summary, Status Reason
- Read-only: Fraud Score, Coverage Status, AI Recommendation, Approval Status

**AI Service Account (Flow Execution):**
- Create: All fields
- Read: All fields
- Update: Fraud Score, Coverage Status, AI Recommendation, Customer Communication, Approval Status, Status Reason

**Manager Role (Supervisor):**
- Read: All fields
- Update: All fields (for override/correction)

---

## Testing Checklist

Before deploying to production, verify all field configurations:

- [ ] All five input fields are created and visible on Case form
- [ ] All four agent output fields are created (read-only on form)
- [ ] Approval Status choice field has three values with correct integer codes
- [ ] Status Reason has three AI-specific choice values (100000001, 100000002, 100000003)
- [ ] Currency field is set to EUR with correct precision
- [ ] Required fields enforce validation at form level
- [ ] Form sections are properly organized and labeled
- [ ] Security roles restrict update permissions appropriately
- [ ] Power Automate flow can successfully update all fields on Case
- [ ] Agent schema output matches field names and types exactly

---

## Troubleshooting

### Field Not Appearing on Form

**Issue:** Created a field but it doesn't show on the Case form.

**Solution:**
1. Open the Case form in the form designer
2. Click **+ Add field**
3. Search for the field logical name (e.g., `cr7ba_fraudscore`)
4. Select it and position it in the desired section
5. Click **Save** and **Publish**

### Power Automate Cannot Update Field

**Issue:** Flow fails when writing to a custom field with "Column does not exist" error.

**Solution:**
1. Verify the logical name in the flow matches exactly (case-sensitive)
2. Confirm the field is in the same solution/environment as the flow
3. Ensure the flow's service account has Update permission on the column
4. Check field data type in flow matches actual field type (e.g., Integer, not String)

### Fraud Score Shows Decimal When Expecting Integer

**Issue:** Field is set as Currency or Decimal instead of Whole Number.

**Solution:**
1. Edit the `cr7ba_fraudscore` column
2. Change Data type to Whole Number (not Currency)
3. Save and republish forms

### Status Reason Value Not Mapping Correctly

**Issue:** Flow writes value 100000001 but Status Reason dropdown shows a different label.

**Solution:**
1. Open the `statuscode` column definition
2. Verify the integer values match your choices exactly
3. Ensure the choice is in the Active (statecode: 0) section, not another state
4. Republish the form to update the dropdown cache

---

## Field Reference Table

Quick lookup table for all fields:

| Display Name | Logical Name | Type | Required | Min | Max | Default |
|---|---|---|---|---|---|---|
| Claim Type | cr7ba_claimtype | Text | Yes | — | 100 | — |
| Claim Amount (Base) | cr7ba_claimamount | Currency | Yes | 0 | 9,999,999.99 | — |
| Claim Summary | cr7ba_claimsummary | Text (Multi) | Yes | — | 2,000 | — |
| Fraud Score | cr7ba_fraudscore | Integer | No | 0 | 100 | — |
| Coverage Status | cr7ba_coveragestatus | Text | No | — | 50 | — |
| AI Recommendation | cr7ba_airecommendation | Text (Multi) | No | — | 1,000 | — |
| Customer Communication | cr7ba_customercommunication | Text (Multi) | No | — | 2,000 | — |
| Approval Status | cr7ba_approvalstatus | Choice | No | — | — | — |
| Status Reason | statuscode | Choice (built-in) | No | — | — | 1 |

---

## Next Steps

After configuring all fields:

1. **Create the Case form** with all fields organized into logical sections
2. **Set up queue rules** that route Cases based on Status Reason
3. **Configure SLA timers** per Status Reason (e.g., 24-hour target for Auto Approved, 48-hour for Pending Approval)
4. **Deploy the Power Automate flow** that populates these fields
5. **Test end-to-end** with sample claims to verify data flow

## Support

If you encounter issues configuring fields, verify:
- You have System Administrator role
- The environment supports custom columns (not a managed solution)
- Field logical names don't conflict with existing fields
- Currency is set to EUR if using currency fields
- Choice values are integers within Dataverse limits

For additional help, consult the [Dynamics 365 Documentation](https://learn.microsoft.com/en-us/dynamics365/customerengagement/on-premises/developer/customize-dev/create-custom-attributes) or contact your Dynamics 365 admin.