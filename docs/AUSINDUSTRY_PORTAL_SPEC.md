# AusIndustry R&D Tax Incentive — Portal submission requirements and field specs

**Linear:** [BEN-35](https://linear.app/ben-g/issue/BEN-35) — Document AusIndustry portal submission requirements and field specs  
**Purpose:** Single source of truth for narrative and submission generator (BEN-36, BEN-37, BEN-38). Defines portal fields, types, character limits, and compliance language for the R&DTI registration application.

---

## 1. Portal overview

| Item | Detail |
|------|--------|
| **Portal URL** | [incentives.business.gov.au](https://incentives.business.gov.au/) (R&DTI customer portal) |
| **Authority** | Department of Industry, Science and Resources (DISR); program administered with ATO |
| **Deadline** | Application must be submitted within **10 months** after end of company income year |
| **Form version** | Updated form mandatory from **15 August 2025** (FY25 and beyond). Previous drafts were removed at cutover. |
| **Definitive guidance** | [Registration – application form questions (PDF)](https://business.gov.au/-/media/grants-and-programs/rdti/rdti-registration-application-form-questions-pdf.pdf) — complete list of questions and character limits. [Registration – application notes (PDF)](https://business.gov.au/-/media/grants-and-programs/rdti/rdti-registration-application-notes-pdf.pdf) for context. |

Portal behaviour:

- **Character limits** are shown at the bottom right of text fields; limits include spaces and line breaks. Some fields have minimum character counts (see official PDF).
- **Mandatory sections** must be completed before other sections unlock: application inclusion pages, contact details, company details, registration type.
- **Dynamic questions** — some questions appear only based on previous answers (e.g. overseas, connected entities).
- **Pre-fill:** Projects/activities can pre-fill from a prior submitted application; activity descriptions still need to be completed.

---

## 2. Character limits (post–August 2025)

As of 15 August 2025, detailed text fields were increased from **1,000 to 4,000 characters** across multiple sections. The table below reflects the updated form; for a complete and authoritative list (including minimums and any differing limits), use the official [Guidance on application questions](https://business.gov.au/grants-and-programs/research-and-development-tax-incentive/apply-for-the-randd-tax-incentive#application-guidance).

| Section | Field / question | Type | Character limit (max) | Notes |
|---------|------------------|------|------------------------|-------|
| **Projects** | How much did the R&D Tax Incentive influence whether you went ahead with this R&D project? | Drop-down | — | Significantly / Somewhat significantly / Not at all |
| **Projects** | Describe what documents you have kept, or intend to keep, in relation to the activities in your project. | Text | 4,000 | Evidence and documentation (contemporaneous records) |
| **Projects** | Is the location of majority of R&D activities the same as the main business address provided? | Radio | — | Yes / No |
| **Projects** | Briefly describe the plant and facilities to be allocated to the project (specialist equipment, facilities etc.) | Text | 4,000 | Plants and facilities |
| **Projects** | To establish whether the company will be a beneficiary … describe whether the company will: effectively own the know-how, IP or other results; have control over direction and conduct of R&D; and bear the financial burden. | Text | 4,000 | “On own behalf” (TA 2023/4, TA 2023/5) |
| **Core activities** | Describe the core R&D activity | Text | 4,000 | Single consolidated description field (replaces multiple shorter fields) |
| **Core activities** | (Unknown outcomes) | Text | 4,000 | Expanded into **two** questions: unknown outcomes; steps to assess existing knowledge |
| **Core activities** | Is the entity who will conduct this core R&D activity a connected or affiliated entity? | Radio | — | Yes / No |
| **Core activities** | Is the core R&D activity being conducted under an agreement with a connected or affiliated entity located outside Australia? | Radio | — | Yes / No |
| **Core activities** | Country of residence for company performing this core R&D activity? | Drop-down | — | |
| **Supporting activities** | (Supporting activity description / link to core) | Text | 4,000 | Dominant purpose test — must support core R&D |
| **Supporting activities** | What evidence did the company keep about this supporting R&D activity? | Text | 4,000 | Evidence and documentation |
| **Supporting activities** | Is the entity who will conduct this supporting R&D activity a connected or affiliated entity? | Radio | — | Yes / No |
| **Supporting activities** | Is the supporting R&D activity being conducted under an agreement with a connected or affiliated entity located outside Australia? | Radio | — | Yes / No |

---

## 3. Compliance language and required concepts

### 3.1 Core R&D activity — systematic progression

Core R&D must be a **systematic progression of work** based on established science. Narrative should reflect:

- **Hypothesis** — proposed explanation for achieving a particular result  
- **Experiment(s)** — scientific procedures to test the hypothesis  
- **Observation** — what happened during experiments  
- **Evaluation** — assessment of results  
- **Logical conclusions** — conclusions drawn from the work  

**Compliance note:** Activities must demonstrate **genuine experimental progression**, not routine or standard development. Outcome must not be knowable in advance; it can only be determined by applying this systematic progression.

**Reference:** [How to conduct eligible R&D activities](https://business.gov.au/grants-and-programs/research-and-development-tax-incentive/assess-if-your-randd-activities-are-eligible/how-to-conduct-eligible-activities); ATO eligibility guidance.

### 3.2 Unknown outcomes and existing knowledge

The updated form splits “unknown outcomes” into **two** questions:

1. **Unknown outcomes** — demonstrate that the outcomes of the R&D could not be known or determined in advance on the basis of current knowledge, information or experience, and could only be determined by applying the systematic progression of work.  
2. **Existing knowledge** — describe the steps taken (upfront) to determine the state of existing knowledge.

Narrative generation (BEN-36) should produce content that maps clearly to both.

### 3.3 “On own behalf”

R&D must be conducted **on the R&D entity’s own behalf**. Narrative and/or structured answers should address (as required by the form):

- **Ownership** — company effectively owns the know-how, intellectual property, or other results arising from the project.  
- **Control** — company has control over the direction and conduct of the R&D activities.  
- **Financial risk** — company bears the financial burden of carrying out the activities.

**Reference:** Taxpayer Alerts TA 2023/4, TA 2023/5 (intercompany and contractual arrangements).

### 3.4 Supporting activities — dominant purpose

Supporting R&D activities must have a **dominant purpose** of supporting core R&D activities. Narrative (BEN-37) must:

- Describe each supporting activity.  
- **Explicitly link** each supporting activity to the core activity it supports.  
- Use language that shows the activity’s purpose is to support core R&D, not production or general business operations.

### 3.5 Evidence and documentation

The form emphasises **contemporaneous record-keeping**. Applicants must describe:

- What documents have been kept, or are intended to be kept, in relation to the project/activities.  
- What evidence was kept for each supporting R&D activity.

Inadequate documentation is a stated compliance risk; generated text should not imply records that do not exist.

---

## 4. Portal sections relevant to narrative generation

| Portal section | Content required | Maps to SubmissionDocument / prompts |
|----------------|------------------|-------------------------------------|
| **Core R&D activity (per activity)** | Title; single consolidated description (4,000 chars) covering what the activity is, scientific/technological uncertainty, systematic progression; two unknown-outcomes questions; entity/overseas questions | `ActivityNarrative`: `title`, `description`, `uncertaintyStatement`, `systematicApproach` (BEN-36) |
| **Supporting R&D activities (per activity)** | Description and explicit link to core activity; evidence kept; entity/overseas questions | `ActivityNarrative` for supporting activities with link to core (BEN-37) |
| **Projects** | Document/evidence kept; plant and facilities; “on own behalf” (beneficiary); R&D location; influence of incentive | Optional narrative or review flags; can be out of scope for initial `generate_submission` and left to accountant |
| **Expenditure by activity** | Which spend lines relate to which registered activity | `ActivityNarrative.linkedExpenditureTotal`, `linkedTransactions`; `FinancialSummary` (BEN-38) |

---

## 5. Mapping to SubmissionDocument and tools

- **BEN-36 (core narrative prompt):** Produce per–core-activity text that fits the **Describe the core R&D activity** field (4,000 chars) and the **two unknown-outcomes** questions. Structure output as `ActivityNarrative` with `title`, `description`, `uncertaintyStatement`, `systematicApproach` so BEN-38 can label and place text into the correct portal fields.  
- **BEN-37 (supporting narrative prompt):** Produce per–supporting-activity text that fits supporting activity description (4,000 chars) and **What evidence did the company keep** (4,000 chars), with explicit link to core activity.  
- **BEN-38 (assemble submission):** Build `SubmissionDocument` with all sections, financial summary, and review flags. Label each narrative block with the corresponding **portal field name** so the accountant can copy into the portal. Flag any items that need human input (e.g. “on own behalf”, plant/facilities) in `reviewFlags`.

---

## 6. References

| Resource | URL / reference |
|----------|------------------|
| Apply for the R&DTI | [business.gov.au/grants-and-programs/.../apply-for-the-randd-tax-incentive](https://business.gov.au/grants-and-programs/research-and-development-tax-incentive/apply-for-the-randd-tax-incentive) |
| Application form questions (PDF) | [rdti-registration-application-form-questions-pdf.pdf](https://business.gov.au/-/media/grants-and-programs/rdti/rdti-registration-application-form-questions-pdf.pdf) |
| Application notes (PDF) | [rdti-registration-application-notes-pdf.pdf](https://business.gov.au/-/media/grants-and-programs/rdti/rdti-registration-application-notes-pdf.pdf) |
| Customer portal help | [business.gov.au/.../customer-portal-help-and-support](https://business.gov.au/grants-and-programs/research-and-development-tax-incentive/customer-portal-help-and-support) |
| How to conduct eligible activities | [business.gov.au/.../how-to-conduct-eligible-activities](https://business.gov.au/grants-and-programs/research-and-development-tax-incentive/assess-if-your-randd-activities-are-eligible/how-to-conduct-eligible-activities) |
| ATO R&D Tax Incentive | [ato.gov.au/.../research-and-development-tax-incentive](https://www.ato.gov.au/businesses-and-organisations/income-deductions-and-concessions/incentives-and-concessions/research-and-development-tax-incentive-and-concessions/research-and-development-tax-incentive) |
| Taxpayer Alerts (on own behalf) | TA 2023/4, TA 2023/5 |
| ATO Tax Ruling (R&D) | TR 2019/1 (eligible R&D activities) |

---

*This document is the single source of truth for portal field specs and compliance language for BEN-36, BEN-37, and BEN-38. When the official form or guidance changes, update this spec and the narrative prompts accordingly.*
