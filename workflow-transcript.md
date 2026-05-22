# Legislative Affairs Flow — Workflow Transcript
*Transcribed from workflow diagram images (workflow-legend.jpg, workflow-step-1.jpg through workflow-step-5.jpg)*
*updated by Sudhakar R after transcription - 5/1/26*
---

## Legend / Key Definitions


User Roles

| Role | Code | Purpose | Typical User |
|------|------|---------|--------------|
| Administrator | `ADMIN` | System administration and emergency overrides | IT Staff, System Admins |
| Legislative Affairs Initiator | `LAI` | Create and initiate bill analyses | Legislative Affairs Staff |
| Administrative Point of Contact | `APOC` | Coordinate bill analysis and review | Designated Contact per Bill |
| Subject Matter Expert | `SME` | Provide expert analysis and review | Department Experts |
| Legislative Affairs Contact | `LA_CONTACT` | First-level approval | LA Department Contacts |
| Senior Deputy | `SENIOR_DEPUTY` | Senior-level approval | Deputy Directors |
| Chief Operating Officer | `COO` | Executive-level approval | Executive Staff |
| Director | `DIRECTOR` | Final approval authority | Department Director |
| Proxy Approver | `PROXY_APPROVER` | Temporary approval authority in absence of primary approver | Designated Proxies |

### Color Coding (diagram shapes)
| Color | Meaning |
|---|---|
| Teal | Auto-generated email |
| Light purple | Configurable approval path |
| Yellow | Review action |
| Green | Approval action |
| Orange | Sub-process |

---

## Full Workflow — Step by Step

### START
**Trigger:** Legislative Proposal needs MDHHS analysis.

---

### Step 1 — LAI Creates and Submits Analysis Record
1. **LAI** creates an analysis record and assigns: APOC, Senior Deputies, LA Contact.
2. **LAI** submits the analysis record inside.
3. *(Auto-generated email)* System sends email notifying: APOC, Senior Deputies, LA Contact.

---

### Step 2 — APOC Facilitates Administration Review
4. **APOC** accesses record to view details.
5. **APOC** assigns SMEs for the record. 
6. **APOC** facilitates administration review and content contribution.
7. **APOC** ensures bill analysis is agreed on by administration SMEs.
8. **APOC** creates the approval path. Approval required from SMEs, Senior Deputies, LA Contact *(Decision/configuration point)*
9. *(Auto-generated email)* System sends email notifying: APOC, LAI.

---

### Step 3 — Administration Review Completion & LAI Review
10. Administration review is confirmed successful and complete.
11. **APOC** accesses record and triggers an automated message to LAI.
12. *(Auto-generated email)* System sends email notifying: APOC, LAI.
13. **LAI** accesses record and reviews the analysis.
14. Executive Approvers and Proxies are added to the approval path. 

---

### Step 4 — LAI Approval & Executive Approval Chain
15. **LAI** approves the analysis.
16. *(Auto-generated email)* System sends email to the **Legislative Affairs Manager** containing the routing sheet for the analysis record.
17. **[PA]**  **COO** Review and Approval is facilitated and completed by **LAI**.*(PA = proxy approver can act on Director's behalf)*
18. **[PA]** **MDHHS Director** approves the analysis. *(PA = proxy approver can act on Director's behalf)*

---

### Step 5 — Final Notification & Artifact Distribution
19. *(Auto-generated email)* System sends email to **LAI** containing:
    - Bill Number
    - Bill Sponsor
    - Bill Analysis Topic
    - Recommended Position
    - Link to bill analysis
20. **LAI** accesses record and clicks the **"Email Artifacts"** button.
21. *(Auto-generated email)* System sends final email containing:
    - PDF file of bill analysis
    - PDF file of routing sheet
    - Bill Analysis details

---

### FINISH

---

## Summary of Auto-Generated Emails

| Trigger Point | Recipients | Contents |
|---|---|---|
| LAI submits analysis record | APOC, Senior Deputies, LA Contact | Analysis record notification |
| APOC creates approval path | APOC, LAI | Approval path created notification |
| APOC triggers message to LAI | APOC, LAI | Administration review complete notification |
| LAI approves analysis | Legislative Affairs Manager | Routing sheet for analysis record |
| Director approves analysis | LAI | Bill #, Sponsor, Topic, Recommended Position|
| LAI clicks "Email Artifacts" | (Recipients TBD) | PDF of bill analysis, PDF of routing sheet, Bill Analysis details |

---

## Roles Involved in This Workflow

| Role | Actions |
|---|---|
| LAI (Legislative Affairs Initiator) | Creates record, assigns roles, submits, reviews, approves, triggers artifact email |
| APOC (Administrative Point of Contact) | Views details, facilitates SME review, builds approval path, triggers LAI notification |
| SMEs (Subject Matter Experts) | Contribute content during administration review |
| Senior Deputies | Assigned at intake; provide input, part of approval path |
| LA Contact (Legislative Affairs Contact) | Assigned at intake; part of approval path |
| Legislative Affairs Manager | Receives routing sheet email after LAI approval |
|  COO | Executive approver (PA-eligible — proxy can act on their behalf) |
| MDHHS Director | Final approver (PA-eligible — proxy can act on their behalf) |

