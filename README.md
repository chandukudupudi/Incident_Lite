# 01 – ITSM Incident Lite

## Project Overview

The ITSM Incident Lite project is a lightweight Incident Management application developed on the ServiceNow platform to simulate a real-world ITSM operational environment. The application demonstrates core ITIL Incident Management capabilities including automatic priority calculation, intelligent assignment routing, SLA management, Record Producer intake, Flow Designer automation, and client-side validations.

The project was designed using a combination of scoped application development and global Incident table customization to reflect enterprise-level ServiceNow implementation practices.

---

# Business Requirement

The organization required a simplified but production-style Incident Management solution capable of:

* Allowing users to report incidents through the Service Portal
* Automatically calculating priority based on Impact and Urgency
* Automatically assigning incidents to support groups and on-call engineers
* Tracking SLA response timelines
* Triggering notifications for major incidents
* Enforcing mandatory fields dynamically
* Improving operational efficiency using automation

---

# Technologies Used

| Technology      | Purpose                          |
| --------------- | -------------------------------- |
| ServiceNow ITSM | Incident Management              |
| Flow Designer   | Workflow Automation              |
| Business Rules  | Server-side automation           |
| Client Scripts  | Client-side validations          |
| Script Includes | Reusable business logic          |
| SLA Definitions | Response time tracking           |
| Record Producer | Service Portal incident creation |
| GlideRecord     | Database operations              |
| Service Portal  | End-user incident intake         |

---

# Application Details

| Item             | Value                              |
| ---------------- | ---------------------------------- |
| Application Name | 01Incident_Lite                    |
| Scope            | Scoped Application                 |
| Main Table Used  | Incident [incident]                |
| Custom Tables    | u_assignment_matrix, u_oncall_rota |

---

# Custom Tables Created

## 1. u_assignment_matrix

Purpose:
Stores routing logic for assignment groups based on Category and Priority.

### Columns

| Field      | Type                       |
| ---------- | -------------------------- |
| u_category | String                     |
| u_priority | Integer                    |
| u_group    | Reference → sys_user_group |

---

## 2. u_oncall_rota

Purpose:
Stores on-call rotation users for round-robin assignment.

### Columns

| Field      | Type      |
| ---------- | --------- |
| u_group    | Reference |
| u_user     | Reference |
| u_sequence | Integer   |

---

# Features Implemented

# 1. Priority Calculation Engine

A reusable Script Include named `PriorityCalc` was developed to dynamically calculate Incident Priority based on Impact and Urgency values.

### Logic

| Impact | Urgency | Priority |
| ------ | ------- | -------- |
| 1      | 1       | P1       |
| 1      | 2       | P2       |
| 2      | 1       | P2       |
| 2      | 2       | P3       |
| 3      | 3       | P5       |

### Script Include Used

```javascript
var PriorityCalc = Class.create();
PriorityCalc.prototype = {

    map: function(impact, urgency) {

        var matrix = {
            '1_1': 1,
            '1_2': 2,
            '1_3': 3,
            '2_1': 2,
            '2_2': 3,
            '2_3': 4,
            '3_1': 3,
            '3_2': 4,
            '3_3': 5
        };

        var key = impact + '_' + urgency;
        return matrix[key] || 3;
    },

    recalcFor: function(inc) {

        var impact = parseInt(inc.impact,10)||3;
        var urgency = parseInt(inc.urgency,10)||3;

        inc.priority = this.map(impact,urgency);
    },

    type: 'PriorityCalc'
};
```

---

# 2. Before Insert Business Rule

Purpose:

* Normalize Incident data
* Set default values
* Calculate priority before record insertion

### Functionalities

* Auto-generates short description if blank
* Defaults Impact/Urgency to Low
* Calls PriorityCalc Script Include

### Business Rule Logic

```javascript
if(!current.short_description){
    short_description = (current.category || 'General') + ' issue reported';
}

if(!current.impact)
    current.impact = 3;

if(!current.urgency)
    current.urgency = 3;

var pc = new incident_lite.PriorityCalc();
pc.recalcFor(current);
```

---

# 3. After Insert Business Rule

Purpose:
Automatically assign incidents using routing matrix and on-call rotation logic.

### Functionalities

* Reads assignment matrix table
* Finds matching support group
* Reads on-call rotation table
* Assigns Incident to support engineer
* Updates Incident automatically

### Business Logic Flow

Incident Created
→ Lookup Category + Priority
→ Find Assignment Group
→ Find Available On-call User
→ Auto Assign Incident

---

# 4. Dynamic Client-Side Validation

An onChange Client Script was implemented on the Category field.

### Requirement

If Category = Network:

* Business Service becomes mandatory
* Configuration Item becomes mandatory

### Script Logic

```javascript
if(g_form.getValue('category')=='network'){

    g_form.setMandatory('business_service',true);
    g_form.setMandatory('cmdb_ci',true);

}else{

    g_form.setMandatory('business_service',false);
    g_form.setMandatory('cmdb_ci',false);
}
```

---

# 5. Record Producer – Report an Issue

A Service Portal Record Producer was developed to allow end users to create incidents easily.

### Variables Created

| Variable    | Type       |
| ----------- | ---------- |
| category    | Select Box |
| subcategory | Select Box |
| urgency     | Choice     |
| impact      | Choice     |
| description | Multi-line |

### Functionalities

* Creates Incident automatically
* Maps portal variables to Incident fields
* Simplifies user interaction

---

# 6. SLA Implementation

An SLA Definition named `Inc_lite_Response` was configured.

### SLA Features

| Feature         | Configuration              |
| --------------- | -------------------------- |
| Type            | SLA                        |
| Table           | Incident                   |
| Duration        | 8 Hours 10 Minutes         |
| Schedule        | 24x7                       |
| Start Condition | State = New OR In Progress |

### SLA Purpose

Tracks Incident response timelines and helps demonstrate enterprise SLA management.

---

# 7. Major Incident Flow Designer Automation

A Flow Designer automation named `Major Incident Alert` was developed.

### Trigger

Incident Created where:

* Priority <= 2

### Actions

1. Create Catalog Task
2. Send Notification

### Business Value

* Enables faster escalation
* Automates major incident communication
* Reduces manual coordination effort

---

# End-to-End Workflow

```text
User submits Incident from Service Portal
        ↓
Record Producer creates Incident
        ↓
Before Business Rule executes
        ↓
Priority automatically calculated
        ↓
After Business Rule executes
        ↓
Assignment Group determined
        ↓
On-call user assigned
        ↓
SLA attached automatically
        ↓
Flow Designer checks Priority
        ↓
Major Incident Notification triggered
```

---

# Real-World Enterprise Concepts Demonstrated

| Concept                  | Implemented |
| ------------------------ | ----------- |
| Incident Lifecycle       | Yes         |
| Dynamic Priority Matrix  | Yes         |
| Assignment Routing       | Yes         |
| On-call Rotation         | Yes         |
| SLA Management           | Yes         |
| Portal Intake            | Yes         |
| Workflow Automation      | Yes         |
| ITSM Best Practices      | Yes         |
| Client-side Validation   | Yes         |
| Reusable Script Includes | Yes         |

---

# Technical Challenges Solved

## Challenge 1

Dynamic priority calculation based on Impact/Urgency.

### Solution

Created reusable Script Include.

---

## Challenge 2

Automatic assignment routing.

### Solution

Created Assignment Matrix and On-call Rota tables.

---

## Challenge 3

Mandatory fields based on Category.

### Solution

Implemented onChange Client Script.

---

## Challenge 4

Major incident communication automation.

### Solution

Implemented Flow Designer notification workflow.

---

# Project Outcome

The Incident Lite application successfully demonstrated a production-style Incident Management lifecycle with automation, SLA tracking, intelligent routing, and portal-based intake.

The project improved:

* Automation efficiency
* Incident routing accuracy
* SLA visibility
* User experience
* Operational consistency


