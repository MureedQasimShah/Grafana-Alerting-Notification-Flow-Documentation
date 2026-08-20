# Grafana Alerting: Notification Flow Documentation

## Overview
This document explains how Grafana's alerting components work together to send notifications when issues are detected.

---

## 1. Component Architecture

```
┌─────────────────┐
│  Alert Rule     │ ──┐
│  (What to       │   │
│   monitor)      │   │
└─────────────────┘   │
                      │
                      ▼
┌─────────────────┐   ┌──────────────────┐
│  Labels &       │──▶│  Notification    │
│  Annotations    │   │  Policy          │
│  (Metadata)     │   │  (Routing Logic) │
└─────────────────┘   └──────────────────┘
                               │
                               ▼
                      ┌──────────────────┐
                      │  Contact Point   │
                      │  (Where to send) │
                      └──────────────────┘
                               │
                               ▼
                      ┌──────────────────┐
                      │  Notification    │
                      │  Template        │
                      │  (Message format)│
                      └──────────────────┘
                               │
                               ▼
                      ┌──────────────────┐
                      │  Email/Slack/etc │
                      │  (Destination)   │
                      └──────────────────┘
```

---

## 2. Component Details & Dependencies

### 2.1 Alert Rule (The Trigger)
**Purpose**: Defines WHAT to monitor and WHEN to alert

**Contains**:
- Query (LogQL, PromQL, etc.)
- Threshold condition (e.g., > 0)
- Evaluation frequency
- **Labels** (for routing)
- **Annotations** (for message content)

**Example**:
```yaml
Name: Migrol 4xx Error Alert
Query: sum by (messagetype) (count_over_time(...))
Condition: > 0
Labels:
  - severity: critical
  - tenant: MIGROL
  - alert_type: invoice_error
Annotations:
  - Description: "Error message details..."
  - dashboard_link: "https://..."
```

**Dependencies**: 
- ✅ Standalone (can exist without other components)
- ❌ Won't send notifications without Contact Point & Notification Policy

---

### 2.2 Labels (The Router)
**Purpose**: Key-value pairs that identify and route alerts

**Used By**:
- Notification Policies (for routing decisions)
- Grouping alerts together
- Filtering in dashboards

**Example**:
```yaml
severity: critical
tenant: MIGROL
alert_type: invoice_error
messagetype: 401
```

**Key Concept**: 
- Labels are **matching criteria** - they determine WHERE alerts go
- Think of them as "routing tags"

---

### 2.3 Annotations (The Message Data)
**Purpose**: Additional metadata for notification content

**Used By**:
- Notification Templates (accessed as variables)
- Alert history and details

**Example**:
```yaml
Description: "Unauthorized - Authentication credentials are missing"
dashboard_link: "https://grafana.com/d/abc123"
summary: "Migrol Invoice Error - 401"
```

**Key Concept**:
- Annotations are **message content** - they provide details for notifications
- Think of them as "message variables"

---

### 2.4 Notification Policy (The Routing Logic)
**Purpose**: Decides WHERE alerts should be sent based on labels

**Contains**:
- **Label matchers** (routing rules)
- **Contact point** assignment
- Grouping rules
- Timing settings

**Example**:
```yaml
Label Matchers:
  - tenant = MIGROL
  - alert_type = invoice_error

Contact Point: Migrol-Email-Alerts

Grouping:
  - Group by: messagetype
  - Group wait: 30s
  - Group interval: 5m
  - Repeat interval: 4h
```

**Dependencies**:
- ⚠️ Requires: Alert Rule (to receive alerts with labels)
- ⚠️ Requires: Contact Point (to know where to send)

**How It Works**:
1. Alert fires with labels: `tenant=MIGROL, alert_type=invoice_error`
2. Policy checks: "Does this alert match my label matchers?"
3. If YES → Send to assigned Contact Point
4. If NO → Check next policy or use default

---

### 2.5 Contact Point (The Destination)
**Purpose**: Defines WHERE and HOW to send notifications

**Contains**:
- Integration type (Email, Slack, PagerDuty, etc.)
- Destination details (email addresses, webhook URLs)
- Optional: Template references

**Example**:
```yaml
Name: Migrol-Email-Alerts
Type: Email
To: support@company.com, helpdesk@company.com
Subject: {{ template "migrol.email.subject" . }}
Message: {{ template "migrol.email.message" . }}
```

**Dependencies**:
- ✅ Can exist standalone
- ⚠️ Needs Notification Policy to receive alerts
- 🔗 Optional: Can reference Notification Templates

---

### 2.6 Notification Template (The Message Format)
**Purpose**: Formats HOW the notification message looks

**Contains**:
- Go template syntax
- References to Labels (`.Labels.xxx`)
- References to Annotations (`.Annotations.xxx`)
- Formatting and styling

**Example**:
```go
{{ define "migrol.email.subject" }}
ERROR ALERT: {{ .CommonLabels.messagetype }}
{{ end }}

{{ define "migrol.email.message" }}
{{ range .Alerts }}
Error Code: {{ .Labels.messagetype }}
Description: {{ .Annotations.Description }}
{{ end }}
{{ end }}
```

**Dependencies**:
- ⚠️ Requires: Alert Rule (to get Labels and Annotations)
- ⚠️ Requires: Contact Point (to be invoked)

**Key Concept**:
- Templates are **optional** - Contact Points can send notifications without them
- Templates access data through: `.Labels.xxx` and `.Annotations.xxx`

---

## 3. Complete Flow Example

### Step-by-Step: How Migrol Alert Works

#### Step 1: Alert Rule Fires
```
Query detects: 15 errors with messagetype=401
Alert fires with:
  Labels:
    - tenant: MIGROL
    - messagetype: 401
    - alert_type: invoice_error
  Annotations:
    - Description: "Unauthorized - Auth failed"
```

#### Step 2: Notification Policy Matches
```
Policy checks labels:
  ✓ tenant = MIGROL → Match!
  ✓ alert_type = invoice_error → Match!
  
Action: Route to Contact Point "Migrol-Email-Alerts"
```

#### Step 3: Contact Point Processes
```
Contact Point "Migrol-Email-Alerts":
  Type: Email
  To: support@company.com
  Subject: {{ template "migrol.email.subject" . }}
  Message: {{ template "migrol.email.message" . }}
  
Action: Load templates and render
```

#### Step 4: Template Renders
```
Template accesses:
  - .Labels.messagetype → "401"
  - .Annotations.Description → "Unauthorized - Auth failed"
  
Renders to:
  Subject: "ERROR ALERT: Issue with Migrol - 401"
  Body: "Error Code: 401
         Description: Unauthorized - Auth failed"
```

#### Step 5: Email Sent
```
Email delivered to: support@company.com
```

---

## 4. Dependency Matrix

| Component | Depends On | Required By | Optional For |
|-----------|------------|-------------|--------------|
| **Alert Rule** | None | All others | - |
| **Labels** | Alert Rule | Notification Policy | - |
| **Annotations** | Alert Rule | Notification Template | Contact Point |
| **Notification Policy** | Alert Rule, Contact Point | Contact Point activation | - |
| **Contact Point** | None | Notification Policy | - |
| **Notification Template** | Alert Rule (for data) | Contact Point (if referenced) | Contact Point (can work without) |

---

## 5. Key Relationships

### Labels vs Annotations

| Aspect | Labels | Annotations |
|--------|--------|-------------|
| **Purpose** | Routing & Filtering | Message Content |
| **Used For** | Matching in policies | Display in notifications |
| **Example** | `tenant=MIGROL` | `Description="Auth failed"` |
| **Indexed** | Yes (searchable) | No |

### Contact Point vs Notification Template

| Aspect | Contact Point | Notification Template |
|--------|---------------|----------------------|
| **Purpose** | WHERE to send | HOW to format |
| **Required** | Yes | No (optional) |
| **Contains** | Email addresses, URLs | Message formatting |
| **Can work alone** | Yes (with default format) | No (needs Contact Point) |

---

## 6. Common Patterns

### Pattern 1: Simple Alert (No Template)
```
Alert Rule → Notification Policy → Contact Point → Email
```
Uses default formatting

### Pattern 2: Custom Formatted Alert (With Template)
```
Alert Rule → Notification Policy → Contact Point → Template → Email
```
Uses custom formatting

### Pattern 3: Multiple Destinations
```
                    → Contact Point (Email) → Template → Email
Alert Rule → Policy → Contact Point (Slack) → Template → Slack
                    → Contact Point (PagerDuty) → Template → Incident
```

---

## 7. Configuration Order

### Recommended Setup Order:

1. **Create Notification Template** (optional, but easier to reference later)
   - Define message formats
   - Create reusable templates for different alert types

2. **Create Contact Point**
   - Add email addresses, Slack channels, or other destinations
   - Reference templates (if created)
   - Test the contact point to verify connectivity

3. **Create Alert Rule**
   - Define query and condition
   - Add labels (for routing)
   - Add annotations (for message content)
   - Set evaluation frequency

4. **Create/Update Notification Policy**
   - Add label matchers
   - Assign contact point
   - Set grouping and timing parameters
   - Order policies by specificity (most specific first)

5. **Test End-to-End**
   - Trigger alert manually or wait for condition
   - Verify routing works correctly
   - Check message format and content
   - Validate all recipients receive notifications

---

## 8. Troubleshooting Guide

| Issue | Check This | Fix |
|-------|------------|-----|
| Alert fires but no notification | Notification Policy | Verify label matchers match alert labels |
| Notification sent to wrong place | Notification Policy | Check policy order & label matchers |
| Message content missing | Annotations | Add annotations to Alert Rule |
| Template error | Template syntax | Check `.Labels` vs `.Annotations` access in correct scope |
| Email not delivered | Contact Point | Test contact point, check SMTP settings |
| Wrong error description | Annotations | Verify annotation template logic and messagetype extraction |
| Multiple duplicate emails | Notification Policy | Check grouping settings (Group by, Group wait) |
| Alert not firing | Alert Rule | Verify query returns data, check threshold condition |

---

## 9. Quick Reference

### To route alerts:
- Use **Labels** in Alert Rule
- Match **Labels** in Notification Policy

### To customize message:
- Add **Annotations** in Alert Rule
- Reference **Annotations** in Template

### To send notifications:
- **Contact Point** = destination (email, Slack, etc.)
- **Notification Template** = format (optional)

### Data flow:
```
Alert Rule (creates data)
    ↓
Labels (route the alert)
    ↓
Notification Policy (makes routing decision)
    ↓
Contact Point (sends notification)
    ↓
Template (formats message using Annotations)
```

---

## 10. Best Practices

### Label Design
- Keep labels simple and consistent
- Use labels for routing, not detailed information
- Common labels: `severity`, `team`, `service`, `environment`
- Avoid high-cardinality labels (e.g., timestamps, UUIDs)

### Annotation Design
- Use annotations for detailed, human-readable information
- Include context that helps with troubleshooting
- Common annotations: `description`, `summary`, `runbook_url`, `dashboard_link`

### Template Design
- Keep templates simple and readable
- Test templates with sample data
- Handle missing data gracefully with conditionals
- Reuse templates across similar alerts

### Notification Policy Design
- Order policies from most specific to least specific
- Use clear, descriptive label matchers
- Set appropriate grouping to avoid notification spam
- Test policy matching with sample alerts

---

## 11. Summary

**Think of it like postal mail:**
- **Alert Rule** = Letter writer (creates the message)
- **Labels** = Address on envelope (routing info)
- **Annotations** = Letter content (actual message)
- **Notification Policy** = Post office sorting (routing decision)
- **Contact Point** = Delivery method (email, courier, etc.)
- **Notification Template** = Envelope & formatting (how it looks)

All components work together, but each has a specific role in getting alerts from detection to notification.

---

## 12. Additional Resources

- **Grafana Alerting Documentation**: https://grafana.com/docs/grafana/latest/alerting/
- **Go Template Documentation**: https://pkg.go.dev/text/template
- **Notification Template Examples**: https://grafana.com/docs/grafana/latest/alerting/manage-notifications/template-notifications/

---

**Document Version**: 1.0  
**Last Updated**: October 2025  
**Author**: Technical Documentation Team
