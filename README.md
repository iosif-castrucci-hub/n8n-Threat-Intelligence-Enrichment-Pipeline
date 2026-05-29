# Threat Intelligence Enrichment Pipeline

A cybersecurity automation project built with **n8n**, **AbuseIPDB**, **VirusTotal**, **Google Sheets**, and **Telegram**.

This project demonstrates an advanced threat intelligence enrichment pipeline that receives an IOC, detects whether it is an IP address, URL, or domain, enriches it using threat intelligence APIs, calculates a risk score, saves the report in Google Sheets, and sends a Telegram alert when the indicator is classified as high risk.

---

## Project Overview

The goal of this project is to automate the enrichment and triage of Indicators of Compromise.

An IOC can be submitted through a Webhook in JSON format. The workflow automatically normalizes the indicator, detects its type, routes it to the correct enrichment branch, collects threat intelligence data, calculates a final risk score, logs the result, and sends an alert if needed.

Supported IOC types:

```txt
IP address
URL
Domain
Unsupported / Unknown IOC
```

This project is designed as a practical demo of **SOAR-style automation**, **threat intelligence enrichment**, **IOC classification**, **risk scoring**, and **security alerting**.

---

## Tech Stack

* **n8n Cloud** — workflow automation platform
* **Webhook Trigger** — receives IOC submissions
* **Code Nodes** — normalize indicators, process enrichment results, calculate risk
* **Switch Node** — routes indicators by type
* **HTTP Request Nodes** — call threat intelligence APIs
* **Wait Node** — waits for VirusTotal URL analysis
* **Google Sheets** — stores enrichment reports
* **IF Node** — checks if the IOC is high risk
* **Telegram Bot API** — sends high-risk threat intelligence alerts
* **AbuseIPDB API** — enriches IP addresses
* **VirusTotal API** — enriches IPs, URLs, and domains

---

## Main Features

### IOC Intake

The workflow starts with a Webhook that receives an IOC.

Example IP submission:

```json
{
  "indicator": "185.220.101.1",
  "source": "Manual Investigation",
  "caseId": "CASE-001",
  "submittedBy": "analyst@example.com"
}
```

Example URL submission:

```json
{
  "indicator": "http://fake-login-security-example.com/verify",
  "source": "Phishing Investigation",
  "caseId": "CASE-002",
  "submittedBy": "analyst@example.com"
}
```

Example domain submission:

```json
{
  "indicator": "example.com",
  "source": "Domain Investigation",
  "caseId": "CASE-003",
  "submittedBy": "analyst@example.com"
}
```

---

### IOC Type Detection

The workflow detects whether the submitted indicator is:

```txt
ip
url
domain
unknown
```

This detection is handled by a Code node before the indicator is routed through the Switch node.

---

### Routing by IOC Type

The workflow uses a Switch node to route indicators into separate enrichment branches.

```txt
IP      → AbuseIPDB + VirusTotal IP Report
URL     → VirusTotal URL Analysis
Domain  → VirusTotal Domain Report
Unknown → Logged as Unsupported
```

---

### IP Enrichment

IP indicators are enriched using:

```txt
AbuseIPDB IP Check
VirusTotal IP Report
```

Collected values include:

```txt
AbuseIPDB Score
AbuseIPDB Total Reports
Country
ISP
Usage Type
VirusTotal Malicious Detections
VirusTotal Suspicious Detections
VirusTotal Harmless Detections
VirusTotal Undetected Detections
```

---

### URL Enrichment

URL indicators are submitted to VirusTotal for analysis.

The workflow:

```txt
Submits URL to VirusTotal
Waits for the analysis
Retrieves the VirusTotal analysis result
Normalizes the enrichment output
```

Collected values include:

```txt
VirusTotal Analysis Status
VirusTotal Malicious Detections
VirusTotal Suspicious Detections
VirusTotal Harmless Detections
VirusTotal Undetected Detections
Extracted Domain
```

---

### Domain Enrichment

Domain indicators are enriched using the VirusTotal Domain Report endpoint.

Collected values include:

```txt
VirusTotal Malicious Detections
VirusTotal Suspicious Detections
VirusTotal Harmless Detections
VirusTotal Undetected Detections
VirusTotal Domain Reputation
VirusTotal Categories
```

---

### Unsupported IOC Handling

If the submitted indicator is not a valid IP address, URL, or domain, the workflow does not fail.

Instead, it logs the indicator as:

```txt
Indicator Type = unknown
Status = Unsupported
Risk Level = Unsupported
```

This keeps the workflow stable and provides traceability for invalid submissions.

---

### Risk Score Calculation

The workflow calculates a final risk score from `0` to `100`.

Risk factors include:

```txt
AbuseIPDB score
VirusTotal malicious detections
VirusTotal suspicious detections
Indicator type
VirusTotal URL analysis status
Negative domain reputation
```

Example scoring logic:

```txt
High AbuseIPDB score → adds risk
VirusTotal malicious detections → adds risk
VirusTotal suspicious detections → adds risk
URL indicator submitted → adds contextual risk
Domain indicator submitted → adds contextual risk
Negative domain reputation → adds risk
```

---

### Risk Level Classification

The workflow classifies the final score as:

```txt
0 - 34     Low
35 - 64    Medium
65 - 84    High
85 - 100   Critical
```

---

### Google Sheets Reporting

Every supported IOC enrichment is saved in Google Sheets.

Unsupported IOCs are also logged.

Google Sheets acts as a lightweight threat intelligence case log.

Stored fields include:

```txt
Report ID
Timestamp
Case ID
Submitted By
Source
Indicator
Indicator Type
AbuseIPDB Score
VT Malicious
VT Suspicious
VT Harmless
VT Undetected
Risk Score
Risk Level
Recommendation
Action Taken
Status
Notes
```

---

### Telegram Alerting

If an IOC is classified as **High** or **Critical**, the workflow sends a Telegram alert.

Example alert:

```txt
🚨 Threat Intelligence Alert

Risk Level: High
Risk Score: 90/100

Indicator: 185.220.101.1
Type: ip

Case ID: CASE-001
Source: Manual Investigation
Submitted By: analyst@example.com

AbuseIPDB Score: 100

VirusTotal:
Malicious: 12
Suspicious: 3
Harmless: 50
Undetected: 20

Recommendation:
Block or monitor the indicator, review related logs, and check for connections from internal assets.

Status: Open
```

After sending the Telegram alert, the workflow updates the Google Sheet row:

```txt
Action Taken = Telegram alert sent
Status = Notified
```

---

## Workflow Structure

The advanced version contains 16 nodes.

```txt
01 Webhook - Receive IOC
02 Code - Normalize and Detect IOC Type
03 Switch - Route by IOC Type

IP Branch:
04A HTTP Request - AbuseIPDB IP Check
05A HTTP Request - VirusTotal IP Report
06A Code - Normalize IP Enrichment

URL Branch:
04B HTTP Request - Submit URL to VirusTotal
05B Wait - Wait for URL Analysis
06B HTTP Request - Get URL Analysis
07B Code - Normalize URL Enrichment

Domain Branch:
04C HTTP Request - VirusTotal Domain Report
05C Code - Normalize Domain Enrichment

Unsupported Branch:
04D Google Sheets - Save Unsupported IOC

Common Branch:
08 Code - Calculate Threat Risk Score
09 Google Sheets - Save Threat Intel Report
10 IF - High Risk Indicator?
11 Telegram - Send Threat Intel Alert
12 Google Sheets - Update Action Taken
```

---

## Workflow Flow

```txt
01 Webhook - Receive IOC
↓
02 Code - Normalize and Detect IOC Type
↓
03 Switch - Route by IOC Type
   ├── IP
   │   ↓
   │   04A HTTP Request - AbuseIPDB IP Check
   │   ↓
   │   05A HTTP Request - VirusTotal IP Report
   │   ↓
   │   06A Code - Normalize IP Enrichment
   │
   ├── URL
   │   ↓
   │   04B HTTP Request - Submit URL to VirusTotal
   │   ↓
   │   05B Wait - Wait for URL Analysis
   │   ↓
   │   06B HTTP Request - Get URL Analysis
   │   ↓
   │   07B Code - Normalize URL Enrichment
   │
   ├── Domain
   │   ↓
   │   04C HTTP Request - VirusTotal Domain Report
   │   ↓
   │   05C Code - Normalize Domain Enrichment
   │
   └── Unknown
       ↓
       04D Google Sheets - Save Unsupported IOC

06A / 07B / 05C
↓
08 Code - Calculate Threat Risk Score
↓
09 Google Sheets - Save Threat Intel Report
↓
10 IF - High Risk Indicator?
   ├── true
   │   ↓
   │   11 Telegram - Send Threat Intel Alert
   │   ↓
   │   12 Google Sheets - Update Action Taken
   └── false
       end
```

---

## Node Details

### 01 Webhook - Receive IOC

Receives the IOC submission.

Configuration:

```txt
HTTP Method: POST
Path: threat-intel-ioc
Authentication: None
Respond: Immediately
```

For production usage, authentication or a secret header should be added.

---

### 02 Code - Normalize and Detect IOC Type

Normalizes the submitted indicator and detects its type.

Output fields include:

```txt
reportId
timestamp
caseId
submittedBy
source
indicator
domain
indicatorType
status
notes
```

Supported types:

```txt
ip
url
domain
unknown
```

---

### 03 Switch - Route by IOC Type

Routes the IOC based on `indicatorType`.

Rules:

```txt
indicatorType = ip      → IP branch
indicatorType = url     → URL branch
indicatorType = domain  → Domain branch
indicatorType = unknown → Unsupported branch
```

---

### 04A HTTP Request - AbuseIPDB IP Check

Checks the IP address reputation using AbuseIPDB.

Configuration:

```txt
Method: GET
URL: https://api.abuseipdb.com/api/v2/check
```

Query parameters:

```txt
ipAddress = {{ $('02 Code - Normalize and Detect IOC Type').item.json.indicator }}
maxAgeInDays = 90
```

Headers:

```txt
Key = YOUR_ABUSEIPDB_API_KEY
Accept = application/json
```

---

### 05A HTTP Request - VirusTotal IP Report

Retrieves the VirusTotal IP report.

Configuration:

```txt
Method: GET
URL: https://www.virustotal.com/api/v3/ip_addresses/{{ $('02 Code - Normalize and Detect IOC Type').item.json.indicator }}
```

Headers:

```txt
x-apikey = YOUR_VIRUSTOTAL_API_KEY
accept = application/json
```

---

### 06A Code - Normalize IP Enrichment

Normalizes AbuseIPDB and VirusTotal IP results into a common format.

Output fields include:

```txt
abuseIpdbScore
vtMalicious
vtSuspicious
vtHarmless
vtUndetected
enrichmentSource
notes
```

---

### 04B HTTP Request - Submit URL to VirusTotal

Submits a URL for VirusTotal analysis.

Configuration:

```txt
Method: POST
URL: https://www.virustotal.com/api/v3/urls
Body Content Type: Form URLencoded
```

Headers:

```txt
x-apikey = YOUR_VIRUSTOTAL_API_KEY
accept = application/json
```

Body:

```txt
url = {{ $('02 Code - Normalize and Detect IOC Type').item.json.indicator }}
```

---

### 05B Wait - Wait for URL Analysis

Waits before retrieving the VirusTotal URL analysis.

Recommended configuration:

```txt
Resume: After Time Interval
Amount: 20
Unit: Seconds
```

If VirusTotal returns `queued`, increase to `30–45 seconds`.

---

### 06B HTTP Request - Get URL Analysis

Retrieves the URL analysis result from VirusTotal.

Configuration:

```txt
Method: GET
URL: https://www.virustotal.com/api/v3/analyses/{{ $('04B HTTP Request - Submit URL to VirusTotal').item.json.data.id }}
```

Headers:

```txt
x-apikey = YOUR_VIRUSTOTAL_API_KEY
accept = application/json
```

---

### 07B Code - Normalize URL Enrichment

Normalizes the VirusTotal URL analysis result.

Output fields include:

```txt
vtStatus
vtMalicious
vtSuspicious
vtHarmless
vtUndetected
enrichmentSource
notes
```

---

### 04C HTTP Request - VirusTotal Domain Report

Retrieves the VirusTotal domain report.

Configuration:

```txt
Method: GET
URL: https://www.virustotal.com/api/v3/domains/{{ $('02 Code - Normalize and Detect IOC Type').item.json.indicator }}
```

Headers:

```txt
x-apikey = YOUR_VIRUSTOTAL_API_KEY
accept = application/json
```

---

### 05C Code - Normalize Domain Enrichment

Normalizes VirusTotal domain results.

Output fields include:

```txt
vtMalicious
vtSuspicious
vtHarmless
vtUndetected
vtDomainReputation
vtCategories
enrichmentSource
notes
```

---

### 04D Google Sheets - Save Unsupported IOC

Saves unsupported indicators in Google Sheets without calling any enrichment API.

Mapped values:

```txt
Risk Score = 0
Risk Level = Unsupported
Recommendation = Submit a valid IP address, URL, or domain for enrichment.
Action Taken = Logged only
Status = Unsupported
```

---

### 08 Code - Calculate Threat Risk Score

Calculates the final threat risk score.

Inputs:

```txt
AbuseIPDB Score
VirusTotal malicious detections
VirusTotal suspicious detections
Indicator type
VirusTotal domain reputation
VirusTotal analysis status
```

Output fields include:

```txt
riskScore
riskLevel
recommendation
actionTaken
status
riskReasons
notes
```

---

### 09 Google Sheets - Save Threat Intel Report

Saves the final enrichment report.

Google Sheet:

```txt
Threat Intel Pipeline
```

Tab:

```txt
Reports
```

---

### 10 IF - High Risk Indicator?

Checks if the final risk level is:

```txt
High
Critical
```

If true, the Telegram alert is sent.

---

### 11 Telegram - Send Threat Intel Alert

Sends a Telegram alert for high-risk or critical indicators.

---

### 12 Google Sheets - Update Action Taken

Updates the same Google Sheets row after the Telegram alert is sent.

Updated fields:

```txt
Action Taken = Telegram alert sent
Status = Notified
```

---

## Google Sheets Structure

Google Sheet:

```txt
Threat Intel Pipeline
```

Tab:

```txt
Reports
```

Columns:

```txt
Report ID
Timestamp
Case ID
Submitted By
Source
Indicator
Indicator Type
AbuseIPDB Score
VT Malicious
VT Suspicious
VT Harmless
VT Undetected
Risk Score
Risk Level
Recommendation
Action Taken
Status
Notes
```

Example row:

| Report ID | Timestamp            | Case ID  | Submitted By                                      | Source               | Indicator     | Indicator Type | AbuseIPDB Score | VT Malicious | VT Suspicious | VT Harmless | VT Undetected | Risk Score | Risk Level | Recommendation                                          | Action Taken        | Status   | Notes                |
| --------- | -------------------- | -------- | ------------------------------------------------- | -------------------- | ------------- | -------------- | --------------: | -----------: | ------------: | ----------: | ------------: | ---------: | ---------- | ------------------------------------------------------- | ------------------- | -------- | -------------------- |
| 128       | 29/05/2026, 10:30:00 | CASE-001 | [analyst@example.com](mailto:analyst@example.com) | Manual Investigation | 185.220.101.1 | ip             |             100 |           12 |             3 |          50 |            20 |         90 | High       | Block or monitor the indicator and review related logs. | Telegram alert sent | Notified | AbuseIPDB score: 100 |

---

## Test Payloads

### IP Test

```json
{
  "indicator": "185.220.101.1",
  "source": "Manual Investigation",
  "caseId": "CASE-001",
  "submittedBy": "analyst@example.com"
}
```

Expected result:

```txt
IOC detected as IP
AbuseIPDB enrichment runs
VirusTotal IP report runs
Risk score is calculated
Google Sheets report is saved
Telegram alert is sent if risk is High or Critical
```

---

### URL Test

```json
{
  "indicator": "http://fake-login-security-example.com/verify",
  "source": "Phishing Investigation",
  "caseId": "CASE-002",
  "submittedBy": "analyst@example.com"
}
```

Expected result:

```txt
IOC detected as URL
VirusTotal URL submission runs
Workflow waits for analysis
VirusTotal URL analysis is retrieved
Risk score is calculated
Google Sheets report is saved
Telegram alert is sent if risk is High or Critical
```

---

### Domain Test

```json
{
  "indicator": "example.com",
  "source": "Domain Investigation",
  "caseId": "CASE-003",
  "submittedBy": "analyst@example.com"
}
```

Expected result:

```txt
IOC detected as Domain
VirusTotal domain report runs
Risk score is calculated
Google Sheets report is saved
Telegram alert is sent if risk is High or Critical
```

---

### Unsupported IOC Test

```json
{
  "indicator": "not-a-valid-ioc",
  "source": "Manual Test",
  "caseId": "CASE-004",
  "submittedBy": "analyst@example.com"
}
```

Expected result:

```txt
IOC detected as unknown
Unsupported IOC is saved in Google Sheets
No enrichment API is called
No Telegram alert is sent
```

---

## cURL Test Examples

### IP Test

```bash
curl -X POST "https://crmsolutions.app.n8n.cloud/webhook-test/threat-intel-ioc" \
-H "Content-Type: application/json" \
-d '{
  "indicator": "185.220.101.1",
  "source": "Manual Investigation",
  "caseId": "CASE-001",
  "submittedBy": "analyst@example.com"
}'
```

---

### URL Test

```bash
curl -X POST "https://crmsolutions.app.n8n.cloud/webhook-test/threat-intel-ioc" \
-H "Content-Type: application/json" \
-d '{
  "indicator": "http://fake-login-security-example.com/verify",
  "source": "Phishing Investigation",
  "caseId": "CASE-002",
  "submittedBy": "analyst@example.com"
}'
```

---

### Domain Test

```bash
curl -X POST "https://crmsolutions.app.n8n.cloud/webhook-test/threat-intel-ioc" \
-H "Content-Type: application/json" \
-d '{
  "indicator": "example.com",
  "source": "Domain Investigation",
  "caseId": "CASE-003",
  "submittedBy": "analyst@example.com"
}'
```

---

### Unsupported IOC Test

```bash
curl -X POST "https://crmsolutions.app.n8n.cloud/webhook-test/threat-intel-ioc" \
-H "Content-Type: application/json" \
-d '{
  "indicator": "not-a-valid-ioc",
  "source": "Manual Test",
  "caseId": "CASE-004",
  "submittedBy": "analyst@example.com"
}'
```

---

## Repository Screenshots

Add your screenshots below.

### IP Enrichment Execution

<!-- Add screenshot of the IP execution here -->

### URL Enrichment Execution

<!-- Add screenshot of the URL execution here -->

### Domain Enrichment Execution

<!-- Add screenshot of the domain execution here -->

### Threat Intel Report Sheet

<!-- Add screenshot of the Google Sheets / Excel report here -->

---

## Security and Business Value

This project demonstrates an advanced threat intelligence enrichment workflow.

It helps security analysts automate repetitive IOC triage tasks by:

* receiving IOCs from external systems or analysts
* detecting IOC type automatically
* enriching IPs, URLs, and domains
* calculating a consistent risk score
* logging results in a central report
* sending alerts only when risk is high
* keeping unsupported submissions traceable

This type of workflow can support:

* SOC automation
* threat intelligence operations
* incident response triage
* phishing investigations
* firewall investigation workflows
* suspicious domain analysis
* case enrichment pipelines

---

## Possible Real-World Use Cases

This workflow can be adapted for:

* SIEM alert enrichment
* phishing email IOC analysis
* firewall alert triage
* suspicious IP/domain review
* SOC analyst manual submissions
* security case enrichment
* incident response investigation support
* automated blocklist review

---

## Future Improvements

Possible upgrades:

* Add URLScan.io enrichment
* Add WHOIS enrichment
* Add DNS resolution enrichment
* Add ASN and geolocation enrichment
* Add CISA KEV or CVE correlation
* Add multiple IOC batch processing
* Add support for file hashes
* Add support for IPv6
* Add support for email indicators
* Add Slack or Microsoft Teams alerts
* Add Jira/Trello/Notion ticket creation
* Add duplicate IOC detection
* Add case status tracking
* Add analyst approval workflow
* Add automatic blocklist update
* Add dashboard reporting
* Add scheduled threat intel summary reports

---

## Security Notes

Before using this in production:

* Protect the Webhook with authentication or a secret token.
* Do not expose API keys in screenshots or GitHub.
* Use n8n credentials or environment variables for secrets.
* Review AbuseIPDB and VirusTotal API rate limits.
* Avoid submitting sensitive internal indicators to third-party APIs without approval.
* Validate incoming payloads.
* Add error handling for API failures.
* Add retry logic for temporary API errors.
* Add deduplication to avoid repeated alerts.
* Protect Google Sheets access permissions.
* Treat this workflow as an enrichment aid, not a complete detection system.

---

## Project Status

Current version: working advanced demo

Implemented:

* Webhook IOC intake
* IOC normalization
* IOC type detection
* IP enrichment branch
* URL enrichment branch
* Domain enrichment branch
* Unsupported IOC logging
* AbuseIPDB integration
* VirusTotal integration
* Threat risk scoring
* Google Sheets reporting
* High/Critical risk IF logic
* Telegram alerting
* Google Sheets status update after alert

---

## Author

Built by **Iosif Castrucci**

GitHub: `iosif-castrucci-hub`
Email: `contact.iosifcastrucci@gmail.com`

---

## Disclaimer

This project is a demo cybersecurity automation workflow created for portfolio and educational purposes.

It is not a complete SOC, SIEM, SOAR, or threat intelligence platform and should not be used as the only method for making security decisions in production environments.

For production use, add authentication, input validation, error handling, deduplication, case management, secure credential storage, audit logging, and a formal incident response process.

---

## License

This repository is intended for portfolio and educational purposes.
You may adapt the workflow structure for your own projects.
