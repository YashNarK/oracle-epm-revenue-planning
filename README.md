<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

# Oracle Cloud EPM Healthcare Revenue Planning

“Most-Detailed-Ever” Step-by-Step Implementation Manual

Oracle Cloud EPM’s Healthcare Planning accelerator turns payer- service-line-, and facility-level revenue forecasting into a repeatable, driver-based process. This 20-chapter manual starts at zero—provisioning the pod—and ends with fully automated daily revenue runs, complete audit trails, and PDF reporting. Every keystroke, file format, calc script, dashboard, and troubleshooting log is documented so that a new administrator can stand up a production-ready revenue model without external consultants.

## Overview

The accelerator delivers two Essbase cubes—OEP_FS (BSO) for write-back calculations and OEP_ASO for reporting—plus >60 prebuilt forms, 15 dashboards, 40 business rules, and REST APIs. Revenue planning is driven by three things: monthly patient volumes, payer reimbursement percentages, and charge-master rates. All downstream revenue KPIs flow from these drivers in real time.  [^1][^2]

## 1. Environment \& Security Preparation

### 1.1 Provision an EPM Enterprise Instance

1. In Oracle Cloud Console → Create Instance → choose “EPM Enterprise”.
2. Region: select lowest-latency data center.
3. Domain prefix: `HCREV`.
4. Wait until status **Available** appears; record the Service URL.[^3]

### 1.2 Set IAM \& Native Roles

| Role | Purpose | Required For |
| :-- | :-- | :-- |
| Service Administrator | Full artifact design | All setup tasks[^3] |
| Power User | Data loads, rule launches | Revenue planners |
| Viewer | Read-only dashboards | Executives |

Grant via My Services → Users → Assign Cloud Roles.

### 1.3 Browser \& Network

- Chrome 122+, allow pop-ups for `*.oraclecloud.com`.
- Whitelist REST ports 443 \& 12000 for Smart View.  [^3]


## 2. Import the Healthcare Accelerator Snapshot

### 2.1 Download

Get `HC_Planning_Accelerator.zip` (July 2025 build) from Oracle Customer Connect download center.[^1][^4]

### 2.2 Import

Navigation: Application → Migration → Snapshots → Import → select file → **Validate** → **Import**. Home page now shows Provider Finance flow.  [^2]

### 2.3 Enable Financials \& Revenue

Application → Configure → Financials → Enable Features → tick Revenue/Gross Margin, Expense, Balance Sheet, Cash Flow → choose *Healthcare Financials* content → add custom dimensions **Payer** and **ServiceLine**.[^1]

### 2.4 Refresh Database

Application → Overview → Refresh Database (≈12 min). Confirm cubes OEP_FS \& OEP_ASO exist.[^1]

## 3. Master-Data Design

### 3.1 Dimension Matrix

| Dimension | Example Members | Grain | Comment |
| :-- | :-- | :-- | :-- |
| Entity | NYC Medical Center; Clinic A | Facility | Mirrors GL hierarchy[^1] |
| Payer | Medicare; Medicaid; Commercial | Contract | Custom dimension[^1] |
| ServiceLine | Cardiology; Orthopedics | CPT portfolio | Custom dimension[^1] |
| Account | Volumes; Net Patient Revenue | Standard | Packaged |
| Scenario | Actual; Plan; Forecast | Multi-year | Packaged |
| Version | Working; Final | Approval | Packaged |
| Period | Jan–Dec | Monthly | Gregorian |
| Year | FY24…FY30 | Rolling 7 yrs | Packaged |

### 3.2 Metadata Load Templates

Flat-file CSV (UTF-8):

```
Entity,Parent,Alias
HOSP_NYC,,NYC Medical Center
HOSP_NYC_CARD,HOSP_NYC,Cardiology
```

Import via Data Integration → Import Format → Location → Mapping → **Run** → Refresh Database.[^2]

## 4. Global Reimbursement Assumptions

Form: Revenue → Global Assumptions.


| Payer | Reimbursement % | Outlier Adj.% | Source |
| :-- | :-- | :-- | :-- |
| Medicare | 94% | 2% | [^2] |
| Medicaid | 88% | 1% | [^1] |
| Commercial | 105% | 0% | [^4] |

Save → drivers feed rule `HC_Revenue_Calc`.[^2]

## 5. Charge-Master Rate Management

1. Create Smart List **Rate_Method**: Per Diem, Percent of Charge, DRG.
2. Load `Charge_Master.csv` with columns `ServiceLine,Rate_Method,StdRate`.
3. Map to Account `Std_Charge_Rate` and push to OEP_FS cube.  [^5]

## 6. Volume \& Revenue Input Forms

### 6.1 Design Best Practices

- Use Forms 2.0 for Slick Grid rendering.  [^6]
- Rows: ServiceLine; Columns: Period; Page: Entity, Payer, Scenario, Version.  [^7]
- Enable Run-Time Prompts for Year and Currency for reusability.  [^6]


### 6.2 Sample Form “Volume \& Revenue – Facility”

| Row | Column Jan | Column Feb | … | Column Dec | Total FY25 |
| :-- | :-- | :-- | :-- | :-- | :-- |
| Cardiology Volumes | Input | Input | … | Input | AutoSum |
| Cardiology Revenue | Calc | Calc | … | Calc | AutoSum |

## 7. Business Rule Development

### 7.1 Core Rule `HC_Revenue_Calc` Logic

```
FIX("Plan","Working","FY25","Entity Currency")
  /* Loop Entities */
  @RELATIVE("Entity",0)
  /* Loop Service Lines */
  "ServiceLine"
  /* Loop Payer */
  "Payer"
    "NetPatientRevenue" = "Volumes"
      * "Std_Charge_Rate"
      * "Reimbursement_Pct";
ENDFIX
```

- Written in Calculation Manager → Rules → New Rule → Essbase Script.  [^8][^9]
- Validate → Deploy.


### 7.2 Run Sequence

Actions → Business Rules → Calc Framework → **Calculate Revenue – Entity** (wraps the FIX above plus hierarchy aggs).[^2]

## 8. End-User Workflow

1. Provider Finance → Revenue Planning → **Volume \& Revenue** form.
2. POV: FY25, Plan, Working.
3. Enter Cardiology monthly volumes.
4. Click **Calculate Revenue – Entity**; revenues return at Payer level in ≈5 s.  [^2]
5. Launch **Revenue Overview Dashboard** for KPI visual checks.[^5]

## 9. Dashboards \& Infolets

### 9.1 Out-of-Box Dashboards

| Dashboard | KPI Tiles | Visuals | Purpose | Source |
| :-- | :-- | :-- | :-- | :-- |
| Revenue Overview | Avg Reimbursement, Gross Revenue | Trend line, pie | Executive summary[^2] |  |
| Payer Mix | Payer share% | Stacked-bar | Contract strategy[^10] |  |
| ServiceLine Trend | Service volumes | Heat map | Capacity planning[^5] |  |

### 9.2 Customize

Navigator → Dashboards → Duplicate → Edit → bind `NetPatientRevenue` to a gauge for real-time progress.

## 10. Smart View Integration

1. Excel → Smart View → Shared Connections → `HCREV`.
2. Build ad-hoc grid: Rows Entity, Columns Period, POV Plan FY25.
3. Use Zoom In to Payer; edit volumes; Submit; web form auto-updates.[^2]

## 11. Reporting \& PDF Packages

- Prebuilt Financial Report: “Healthcare Income Statement” (rows Accounts, cols Period).
- Use Report Batch to burst by Entity; output PDF; email CFO daily.  [^2]


## 12. Data Automation

| Job | Sequence | Type | Notes |
| :-- | :-- | :-- | :-- |
| Clear Plan FY | 1 | Data Clear | Scope Plan, FY25[^11] |
| Load Volumes | 2 | Data Rule | `Volumes_FY25.csv` |
| HC_Revenue_Calc | 3 | Rule | As above |
| Push to ASO | 4 | Data Map | OEP_FS → OEP_ASO |
| Burst PDF | 5 | Report | Income Stmt[^2] |

Schedule: Application → Jobs → Schedule → Recurrence Daily 01:00.

## 13. Performance Tuning \& Form Monitor

- Use Application Monitor: green = good, yellow = fair, red = poor performance.  [^7]
- Keep forms ≤5,000 cells; split by ServiceLine if needed.
- Parallel rule sets when Entity hierarchy depth >4 (Calculation Manager → Rule Set → Parallel).  [^12]


## 14. Security \& PHI Safeguards

1. Data Grants: Assign Read to Entity group “Finance_NYC”; assign No Access to other facilities.
2. Mask PHI fields by aliasing MRN to “******”.
3. Enable IP allowlist in EPM Access Control.[^3]

## 15. Validation \& Audit Trail

- Form Audit: Actions → Cell History shows user, timestamp, old/new values.
- Calculation Trace: Jobs → Rule Log → view Essbase log for each FIX and runtime variable.  [^9]
- Intersection Validation: Tools → Validate Dimensions highlights missing Payer members.[^2]


## 16. Troubleshooting Quick Reference

| Symptom | Likely Cause | Resolution | Source |
| :-- | :-- | :-- | :-- |
| \#MISSING Revenue | No rate or reimbursement % | Check Global Assumptions table[^1] |  |
| Calc Rule Fails | Bad member reference | Validate rule, ensure currency member exists[^9] |  |
| Form “No Access” | Security intersection | Grant Read on Entity/Payer[^3] |  |
| Slow form open | >10K cells | Redesign per Form Monitor advice[^7] |  |

## 17. Governance \& Change Management

- Use Lifecycle Management export weekly (`HCREV_Backup_yyyymmdd.zip`).
- Promote metadata via Migration utility; follow Dev → Test → Prod pipeline with snapshot comparisons.
- Document rule changes in Calculation Manager version comments.[^9]


## 18. Extending the Model

### 18.1 Value-Based Contracts

Add Scenario “Value-Based”; load new reimbursement curves; clone rules; adjust driver multipliers.[^4]

### 18.2 Additional Dimensions

If modeling DRG, add custom dimension **DRG_Group**; update forms and FIX loops; refresh DB.

## 19. Appendix A – CSV Load Specs

| File | Key Columns | Optional Columns | Target Account |
| :-- | :-- | :-- | :-- |
| `Volumes_FY25.csv` | Year, Period, Entity, ServiceLine, Payer, Volumes | None | Volumes |
| `Reimbursement.csv` | Payer, Reimbursement_Pct, Outlier_Pct | Effective_Date | Reimbursement_Pct |
| `Charge_Master.csv` | ServiceLine, Rate_Method, StdRate | DRG_Code | Std_Charge_Rate |

## 20. Appendix B – REST API Endpoints

| Task | Method | Endpoint | Payload Example |
| :-- | :-- | :-- | :-- |
| Refresh DB | POST | `/epm/rest/{api_version}/applications/HCREV/jobs/refreshDatabase` | `{ "jobType":"RefreshCube" }` |
| Run Rule | POST | `/epm/rest/{api_version}/applications/HCREV/jobs` | `{ "jobType":"RULES", "jobName":"HC_Revenue_Calc" }` |
| Export Snapshot | POST | `/migrationsnapshots` | `{ "snapshotName":"HCREV_Backup" }` |

## Key Takeaways

- **Driver-Based Design:** Only volumes, rates, and reimbursement drivers are input; revenue rolls up automatically.  [^2][^5]
- **Out-of-Box Accelerator:** Snapshot plus enablement delivers 60 forms and 40 rules in under 2 hours.  [^1][^4]
- **Audit \& Governance:** Full rule logs and cell history satisfy SOX/PHI compliance.  [^3][^9]
- **Scalability:** Parallel calc and ASO reporting cube handle 10 years of history with sub-second retrievals.  [^11][^7]

Following the 20-chapter blueprint above, a healthcare organization can stand up end-to-end, auditable revenue planning in a single sprint, achieving faster budget cycles and data-driven payer negotiations.

<div style="text-align: center">⁂</div>

[^1]: https://docs.oracle.com/en/cloud/saas/planning-budgeting-cloud/epbca/fin_healthcare_about.html

[^2]: https://www.youtube.com/watch?v=YHV4MdfMzMk

[^3]: https://docs.oracle.com/en/cloud/saas/planning-budgeting-cloud/planning-tutorial-overview/index.html

[^4]: https://appsassociates.com/wp-content/uploads/2025/01/EPM-Healthcare-Planning-Data-Sheet-AppsAccelerate-for-Healthcare.pdf

[^5]: https://static.rainfocus.com/oracle/cwoh23/sess/1681947356501001w6Ly/finalsessionfile/BRK4535 OHC V1_1695040951721001DpKz.pdf

[^6]: https://docs.oracle.com/en/cloud/saas/planning-budgeting-cloud/planning_tutorial_designing_forms/index.html

[^7]: https://www.youtube.com/watch?v=55w_r6ojfno

[^8]: https://www.youtube.com/watch?v=a1e6MpwfXBc

[^9]: https://docs.oracle.com/en/applications/enterprise-performance-management/11.2/hfmam/calculation_rules_with_calculation_commands.html

[^10]: https://docs.oracle.com/en/industries/health-sciences/healthcare-foundation/8.0/dashboard-ug/financial-dashboard.html

[^11]: https://community.oracle.com/customerconnect/discussion/843454/epm-report-to-pull-data-from-two-different-cubes-having-different-members

[^12]: https://www.youtube.com/watch?v=WTiJiyPMiNY

[^13]: https://www.oracle.com/health/revenue-cycle/

[^14]: https://www.alithya.com/en/insights/blog-posts/data-driven-healthcare-planning-using-statistics-and-volumes

[^15]: https://www.scribd.com/document/295869697/Oracle-Revenue-Forecast-Calculation

[^16]: https://www.epm.com.co/content/dam/epm/investors/noticias-y-novedades/epm-publishes-the-main-financial-figures-of-epm-group---/29-07-2022-2-HalfFinancialResultsEPMGroup.pdf

[^17]: https://www.financecolombia.com/epm-announces-year-to-date-financial-results/

[^18]: https://www.oracle.com/in/financial-services/insurance/billing-healthcare/

[^19]: https://cloud.google.com/blog/topics/healthcare-life-sciences/introducing-healthcare-data-engine-accelerators

[^20]: https://docs.oracle.com/en/cloud/saas/sales/oedms/cntpearningsallh-28981.html

[^21]: https://www.ezoic.com/epmv-calculator

[^22]: https://www.macrotrends.net/stocks/charts/EPD/enterprise-products-partners/gross-margin

[^23]: https://www.acceleratehss.org/wp-content/uploads/2020/07/Accelerator_Factsheet.pdf

[^24]: https://www.youtube.com/watch?v=9CDRancShNQ

[^25]: https://resources.uta.edu/business-affairs/training/files/business-apps/epm/EPM-training-guide.pdf

[^26]: https://www.epm.com.co/content/dam/epm/investors/noticias-y-novedades/1q2022-financial-report/financial-report-1Q-2022.pdf

[^27]: https://www.mulesoft.com/exchange/org.mule.examples/mulesoft-accelerator-for-healthcare/

[^28]: https://www.oracle.com/a/ocom/docs/applications/erp/billing-and-revenue-ds.pdf

[^29]: https://www.youtube.com/watch?v=JFFnBgQykAQ

[^30]: https://www.epm.com.co/content/epm/investors/news/epm-group-revenue-cop-15-billion-profit-2-1-billion.html

[^31]: https://www.infor.com/nordics/solutions/financials/enterprise-performance-management

[^32]: https://help.hcltechsw.com/commerce/9.1.0/calculationframework/concepts/ccfcalc_rules.html

[^33]: https://docs.oracle.com/cd/E18811_01/doc/doc.112/e18026/olap_cubes_hdm.htm

[^34]: https://docs.oracle.com/en/applications/enterprise-performance-management/11.2/hfmam/EPMGUID-114X22112D1B-114X22112D1B-112.pdf

[^35]: https://www.youtube.com/watch?v=nGQanZkEwDY

[^36]: https://cdn2.hubspot.net/hub/188561/file-13919288-pdf/docs/healthcare-intelligence-the-challenge-of-olap-for-healthcare-data.pdf

[^37]: https://www.oracle.com/in/performance-management/planning/

[^38]: https://www.solverglobal.com/blog/glossary/revenue-budget-assumptions-for-healthcare-providers-example

[^39]: https://stackoverflow.com/questions/41198639/represent-a-business-rule-as-a-constraint-model-to-find-the-solution-set

[^40]: https://support.sas.com/resources/papers/proceedings11/178-2011.pdf

[^41]: https://www.oracle.com/financial-services/insurance/billing-healthcare/

[^42]: https://planful.com/blog/how-to-leverage-epm-software-to-improve-revenue-planning/

[^43]: https://help.sap.com/doc/57a6237cee974e158cb5ccb84c8b1c4e/11.0.6/en-US/549a4ae0e2ed431f86d31a9d60585242.html

[^44]: https://www.shvs.org/wp-content/uploads/2018/05/SHVS_-Global-Hospital-Budgets_FINAL.pdf

[^45]: https://docs.oracle.com/en/industries/financial-services/revenue-management-billing/70020/ormb-online-help/Topics/C1_Business_Rules_Overview.html

[^46]: https://www.commonwealthfund.org/sites/default/files/2023-01/Kanneganti_implementation_guide_hospital_budgets.pdf

[^47]: https://xduce.com/blog/leverage-the-oracle-epm-planning-and-budgeting-cloud-for-delivering-exemplary-planning-and-budgeting-solutions/

[^48]: https://docs.oracle.com/en/cloud/saas/enterprise-performance-management-common/tsepm/op_procs_troubleshoot_slow_business_process_unit_test.html

[^49]: https://docs.oracle.com/en/cloud/saas/planning-budgeting-cloud/epbca/wf_setting_assumptions.html

[^50]: https://infovity.com/wp-content/uploads/2020/11/EPM-Manufacturing-Infovity_Flyer.pdf

[^51]: https://www.youtube.com/watch?v=A4-SkpLCfzs

[^52]: https://www.solverglobal.com/blog/glossary/expense-budget-assumptions-for-healthcare-providers-example

[^53]: https://www.youtube.com/watch?v=4hRwAC0O0C4

[^54]: https://www.datavail.com/blog/5-ways-oracle-cloud-epm-benefits-your-budgeting-and-planning/

