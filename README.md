# SOAR Pipeline — Microsoft Sentinel to Jira

### A Sentinel brute-force incident creates a Jira ticket automatically, with the incident details already mapped into the fields

<p align="left">
  <img src="https://img.shields.io/badge/Microsoft_Sentinel-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white"/>
  <img src="https://img.shields.io/badge/Azure_Logic_Apps-5C2D91?style=for-the-badge&logo=azure&logoColor=white"/>
  <img src="https://img.shields.io/badge/Jira-0052CC?style=for-the-badge&logo=jira&logoColor=white"/>
  <img src="https://img.shields.io/badge/KQL-00B4D8?style=for-the-badge&logo=microsoftazure&logoColor=white"/>
  <img src="https://img.shields.io/badge/MITRE_ATT%26CK-T1110-E63946?style=for-the-badge&logo=shield&logoColor=white"/>
</p>

---

## What this does

| Step | What happens | Where |
|------|-------------|-------|
| 1 | An attacker script generates rapid Entra ID sign-in failures | Attacker |
| 2 | A KQL analytic rule matches the brute-force pattern (error `50126`) | Microsoft Sentinel |
| 3 | A High-severity incident fires; an automation rule triggers | Microsoft Sentinel |
| 4 | A Logic App playbook runs | Azure |
| 5 | An HTTP POST with the incident details hits the Jira REST API | Logic App |
| 6 | A Jira ticket appears in the SOC queue, fields already filled in | Jira |

The detection query, the Jira request body, and the full Logic App definition are committed in [`queries/`](queries/) and [`playbook/`](playbook/).

---

## What This Project Is

Copying alert details from a SIEM into a ticketing queue by hand is slow and error-prone, and it's time an analyst should spend on triage rather than data entry. This project automates that hand-off: **Microsoft Sentinel** detects a brute-force attack against **Entra ID**, and an **Azure Logic App** creates a **Jira** ticket through the REST API with the incident's details already mapped into the issue — no manual step between detection and ticket.

I built it as a lab to understand how SOAR automation reduces mean time to respond (MTTR), and where the practical friction is: trigger choice, credential handling, RBAC, and the JSON depth of the Jira API.

---

## Architecture

```mermaid
graph LR
    A["Attacker Script<br/>Brute-force auth loop"] -->|Rapid auth failures| B["Microsoft Entra ID"]
    B -->|Auth telemetry| C["Log Analytics Workspace"]
    C -->|"KQL rule<br/>ResultType 50126"| D["Microsoft Sentinel<br/>High-severity incident"]
    D -->|Automation rule triggers| E["Azure Logic App<br/>playbook-sentinel-to-jira"]
    E -->|"HTTP POST<br/>JSON body"| F["Atlassian Jira<br/>SOC ticket queue"]
```

---

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **SIEM** | Microsoft Sentinel | Detection + incident management |
| **Identity Provider** | Microsoft Entra ID | Auth telemetry source |
| **Log Storage** | Log Analytics Workspace | Event ingestion + KQL |
| **Automation** | Azure Logic Apps | Playbook execution |
| **ITSM** | Atlassian Jira | Ticket queue destination |
| **Integration** | Jira REST API v3 + HTTP action | Cross-platform ticket creation |
| **Detection Language** | KQL | Brute-force analytic rule — [`queries/`](queries/) |
| **MITRE ATT&CK** | T1110 — Brute Force | Detection mapping |

---

## Build Phases

### Phase 1 — Attack Simulation (the trigger)

To generate real telemetry, I ran a PowerShell authentication loop against a test Entra ID account — a rapid sequence of wrong passwords against a test account, which is what an automated password-guessing tool produces. This populated the sign-in logs with the failure pattern the detection is built to catch.

![Attack Simulation](images/00-attack-simulation-brute-force.png)

---

### Phase 2 — Cloud Infrastructure & Log Storage

A dedicated resource group in Azure, with a Log Analytics Workspace to ingest and store the incoming sign-in logs.

![Resource Group Creation](images/02-resource-group-creation-form.png)

![Resource Group Deployed](images/01-resource-group-created.png)

![Log Analytics Workspace](images/03-log-analytics-workspace-created.png)

---

### Phase 3 — Sentinel & Data Connectors

Microsoft Sentinel enabled on the workspace, with the native Entra ID connector routing sign-in and audit logs into it.

![Sentinel Enabled](images/04-microsoft-sentinel-enabled.png)

![Defender Connected](images/05-sentinel-workspace-connected-to-defender.png)

![Entra ID Connector](images/06-defender-data-connectors-entra-visible.png)

![Connector Config](images/07-entra-id-connector-configuration.png)

![Ingestion Confirmed](images/08-entra-data-ingestion-confirmed.png)

![Audit Logs Query](images/09-auditlogs-query-results.png)

---

### Phase 4 — Detection Rule (KQL)

The analytic rule filters sign-in failures down to error code `50126` (invalid username or password) — the exact signal a brute-force attack against Entra ID produces. Filtering on this code keeps MFA failures and conditional-access blocks out of the alert scope. Full query: [`queries/failed-signin-bruteforce.kql`](queries/failed-signin-bruteforce.kql).

```kusto
SigninLogs
| where ResultType == "50126"          // invalid username or password — the brute-force signal
| where TimeGenerated > ago(1h)        // interactive runs only - the scheduled rule uses its own lookback
| summarize FailedAttempts = count() by UserPrincipalName, IPAddress, Location
| where FailedAttempts > 5             // below Entra smart lockout's default of 10, so the rule fires while failures still return 50126
| project UserPrincipalName, IPAddress, Location, FailedAttempts
| order by FailedAttempts desc
```

Two deployment details worth knowing: the `ago(1h)` line is for running the query interactively (as in the screenshot below) — in the scheduled rule the lookback is set in the rule configuration instead, because hardcoding both can miss events when ingestion lags. And the rule's entity mapping ties `IPAddress` to the IP entity and `UserPrincipalName` to the Account entity, which is what the playbook later reads from the incident.

![Live KQL Results](images/10-failed-signin-kql-live-results.png)

![Analytics Rule Created](images/11-analytics-rule-created.png)

![Rule Simulation](images/12-analytics-rule-simulation-results.png)

---

### Phase 5 — Jira Setup

Before automating the response, the Jira side needed a project and an API token so the Logic App could authenticate to the REST API.

![Jira Project Created](images/13-jira-project-created.png)

![Jira API Token](images/14-jira-api-token-created.png)

---

### Phase 6 — Logic App Playbook

The playbook is two nodes: a **Microsoft Sentinel incident** trigger, and an **HTTP POST** to the Jira REST API. The full definition is committed at [`playbook/logic-app-workflow.json`](playbook/logic-app-workflow.json); the Jira request body is at [`playbook/jira-issue-body.json`](playbook/jira-issue-body.json).

Three things worth calling out:

- I used the **incident trigger**, not the alert trigger. The incident trigger carries the full aggregated context — all entities, all alerts, mapped tactics — where the alert trigger only carries a single alert. A ticket built from the incident is actually actionable.
- Jira's REST API v3 uses **Atlassian Document Format (ADF)** for the description — a deeply nested JSON structure — so the Sentinel dynamic fields (`title`, `severity`) are mapped into that ADF body rather than a plain string.
- The Jira credentials are encoded as **Base64 Basic Auth** in the HTTP header. That's fine for a lab; in production these would move to **Azure Key Vault** and be read via a **Managed Identity** so nothing is hardcoded.

![Logic App Created](images/15-logic-app-playbook-created.png)

![Incident Trigger](images/16-sentinel-incident-trigger-configured.png)

![API Authorized](images/18-azuresentinel-api-connection-authorized.png)

![HTTP Action Config](images/17-logic-app-http-action-jira-configured.png)

---

### Phase 7 — RBAC & Automation Rule

The Logic App needs the **Microsoft Sentinel Responder** role on the workspace to read incident data — without it the playbook fires but returns 403 on the data fetch. A Sentinel automation rule wires the analytic rule to the playbook so it runs the moment an incident is created.

![Automation Rule](images/19-sentinel-automation-rule-created.png)

![RBAC Permissions](images/20-sentinel-logic-app-permission-granted.png)

![Pipeline Active](images/24-automation-rule-active-confirmed.png)

---

### Phase 8 — End-to-End Test

Re-running the brute-force script drove the full pipeline with no manual steps. The Jira API returned **HTTP 201 Created**, and the ticket landed in the SOC queue with its fields populated.

![Sentinel Incident](images/21-sentinel-incidents-created.png)

![Playbook Triggered](images/22-playbook-triggered-successfully.png)

![API 201 Created](images/22b-logic-app-run-history-success.png)

![Jira Ticket Created](images/23-jira-ticket-auto-created.png)

![Ticket Details Mapped](images/23b-jira-ticket-details-mapped.png)

---

## Key Decisions & Notes

### Incident trigger vs alert trigger
The incident trigger was the right call because the ticket needs the full picture — all entities and tactics — not one alert's fields. This is the difference between a ticket an analyst can act on and one they have to go back to Sentinel to understand.

### Failure handling
As run in the lab, the playbook did a single POST with Logic Apps' default behaviour — a Jira outage or rate-limit would have silently dropped the ticket. The committed definition now carries an explicit exponential `retryPolicy` (4 attempts from a 20-second base), added after the lab run. The remaining gap is a fallback when every retry fails — an email or Teams message to the SOC channel — so a dropped ticket is visible rather than lost.

### Ticket depth
The ticket body is minimal: title and severity. An analyst opening it still has to go to Sentinel for the attacker IP and target account, which costs back some of the time the automation saves. The upgrade is parsing the incident's entities — the Sentinel connector's "Entities - Get IPs" and account actions — into the ADF description so the indicators land in the ticket itself. I've kept the committed workflow faithful to what ran rather than adding untested entity-parsing steps.

### Credentials
Base64 Basic Auth in the header is acceptable for a lab but not for production. The production version moves the token to Azure Key Vault and reads it via a Managed Identity, so the secret never lives in the workflow definition. The committed `logic-app-workflow.json` has the auth header, Jira domain, and connection id redacted.

### What this is worth
In a manual workflow an analyst has to notice the alert, open Sentinel, read the incident, open Jira, and copy everything across. This pipeline removes those steps for the detection-to-ticket hand-off, which is the slice of MTTR that's pure overhead.

---

## What's Next

- **Bi-directional sync** — update the Sentinel incident status when the Jira ticket is resolved
- **Entity-rich tickets** — parse the incident's entities (attacker IP, target account) into the Jira description via the connector's entity actions, then enrich with AbuseIPDB or VirusTotal reputation scores
- **Fallback notification** — a Teams or email alert when all HTTP retries fail (the retry policy itself is now in the committed definition)

---

## Repository Structure

```
azure-sentinel-jira-soar-pipeline/
│
├── README.md
├── LICENSE
├── .gitignore
│
├── queries/
│   └── failed-signin-bruteforce.kql    # the Sentinel analytic rule (ResultType 50126)
│
├── playbook/
│   ├── logic-app-workflow.json         # full Logic App definition (secrets redacted)
│   └── jira-issue-body.json            # the ADF request body sent to the Jira REST API
│
└── images/                              # 27 screenshots — build, detection, playbook, end-to-end test
```

---

## Related Projects

| Project | Description |
|---------|-------------|
| [Dual SIEM Detection Lab](https://github.com/shank078/Dual-SIEM-Detection-Lab) | 5 MITRE-mapped detections across Sentinel + Splunk on live attacker traffic |
| [Azure Sentinel Honeypot SIEM](https://github.com/shank078/azure-sentinel-honeypot-siem) | 1,400+ real brute-force attempts captured and mapped globally |
| [Azure Identity Security Lab](https://github.com/shank078/azure-identity-security-lab) | Red/blue team MFA compromise and IR cycle |

---

## About

Built and documented by **Shankar Baral** — junior SOC analyst in Canberra, Australia. More about me and my other labs: [github.com/shank078](https://github.com/shank078) · [LinkedIn](https://www.linkedin.com/in/shankarbaral1)
