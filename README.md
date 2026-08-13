# Insurance Quote-to-Bind Analytics — Power BI Portfolio Project
### Synthetic Dataset Documentation

This project simulates an India-based insurance operations team processing property quote requests submitted by Australian brokers on behalf of an Australian Lloyd's-market MGA client, from initial broker request through quality review, underwriting, quote delivery, and binding. All data is entirely fictional and safe to publish.

**Data period:** 1 June 2024 – 31 May 2026 (24 months)

**Final table volumes:**

| Table | Rows |
|---|---|
| FactQuote | 42,035 |
| FactQualityReview | 44,201 |
| FactUnderwriting | 35,485 |
| FactBinding | 35,067 (13,073 bound → **binding rate 37.3%**) |
| DimBroker | 42 |
| DimEmployee | 55 |
| DimProperty | 104 |
| DimLocation | 18 |
| DimErrorCategory | 15 |
| DimReferralReason | 12 |
| DimStatus | 10 |
| DimDate | 730 |

All foreign keys verified — every fact-table FK resolves to an existing dimension/fact row, no negative premiums, no broken relationships.

---

## 1. Star Schema & Relationships

**Fact tables:** FactQuote, FactQualityReview, FactUnderwriting, FactBinding
**Dimension tables:** DimDate, DimBroker, DimEmployee, DimProperty, DimLocation, DimErrorCategory, DimReferralReason, DimStatus

| From (1) | To (many) | Cardinality | Cross-filter |
|---|---|---|---|
| DimDate[DateKey] | FactQuote[ReceivedDateKey] | 1:M | Single — **active relationship** |
| DimBroker[BrokerID] | FactQuote[BrokerID] | 1:M | Single |
| DimEmployee[EmployeeID] | FactQuote[QuotingAgentID] | 1:M | Single |
| DimEmployee[EmployeeID] | FactQualityReview[QuotingAgentID] | 1:M | Single |
| DimEmployee[EmployeeID] | FactQualityReview[ReviewerID] | 1:M | Single (inactive if ambiguous — set active per report page need) |
| DimEmployee[EmployeeID] | FactUnderwriting[UnderwriterID] | 1:M | Single |
| DimProperty[PropertyProfileKey] | FactQuote[PropertyProfileKey] | 1:M | Single |
| DimLocation[LocationKey] | FactQuote[LocationKey] | 1:M | Single |
| DimErrorCategory[ErrorCategoryID] | FactQualityReview[ErrorCategoryID] | 1:M | Single |
| DimReferralReason[ReferralReasonID] | FactUnderwriting[ReferralReasonID] | 1:M | Single |
| DimStatus[StatusID] | FactBinding[StatusID] | 1:M | Single |
| FactQuote[QuoteID] | FactQualityReview[QuoteID] | 1:M | Single (used for drillthrough/relationships between facts) |
| FactQuote[QuoteID] | FactUnderwriting[QuoteID] | 1:1 (mostly) | Single |
| FactUnderwriting[QuoteID] | FactBinding[QuoteID] | 1:1 | Single |

**Note on DimEmployee:** because it's used three times (agent, reviewer, underwriter), Power BI will only let one relationship be active by default. Keep `DimEmployee → FactQuote[QuotingAgentID]` active, and use `USERELATIONSHIP()` in DAX measures for reviewer- and underwriter-specific analysis, or create role-specific bridge tables if you want all three simultaneously filterable from slicers (recommended for the Employee Performance page).

**Note on DimDate role-playing:** only `ReceivedDateKey` has a formal DateKey relationship. Other stage timestamps (assigned, rater completed, underwriting date, QSTB date, binding date) live as datetime columns directly on the fact rows — calculate durations from those directly rather than adding 6+ additional DimDate relationships, which would create unnecessary model complexity.

---

## 2. Data Generation Logic Summary

Generation happened in dependency order so correlations flow naturally downstream:

1. **Reference tables first** — brokers and employees were each assigned internal (non-exported) performance profiles: broker size → volume weight, broker → conversion-rate multiplier and request-quality multiplier; employee experience level → error-rate and speed multipliers, plus individual noise so no two employees perform identically.
2. **FactQuote** — property/risk fields drawn per broker's home state (state-level flood/bushfire baselines), with a composite `RiskScore` computed from building value, construction/occupancy risk tier, claims history, and location flags. Monthly volume follows a seasonal curve (AU-winter dip, Oct–Dec and Mar–May uplift) plus a mild 24-month growth trend, so volume trends are not flat.
3. **FactQualityReview** — error probability is a function of the assigned agent's error multiplier × the submitting broker's quality multiplier × a workload-pressure term (busier agents skew slightly higher error rate). Rework triggers a second review pass with a much lower repeat-error probability.
4. **FactUnderwriting** — premium = (sum insured ÷ 1000) × base rate (residential vs commercial) × risk multiplier × claims loading × noise (±10–15%), so premium correlates with building value, risk score, construction, and claims without being deterministic. Referral probability is a logistic function of premium size, risk score, flood/bushfire flags, and claims count — high-value, high-risk quotes refer to Australia noticeably more often (~28% overall referral rate).
5. **FactBinding** — bind probability combines a premium-range base conversion curve (cheaper quotes convert better) with each broker's individual conversion multiplier, so some brokers consistently outperform others at the same premium level.

---

## 3. Intentional Data-Quality Issues (for Power Query practice)

These were injected deliberately, on top of the clean logical dataset, so they don't corrupt underlying business relationships — only the surface presentation:

| Table | Issue | Count | Fix in Power Query |
|---|---|---|---|
| DimBroker | Leading/trailing whitespace and inconsistent case in `BrokerName` | 6 rows | `Text.Trim`, `Text.Proper` |
| DimBroker | Lowercased `State` values | 3 rows | `Text.Upper` or a state-code merge |
| FactQuote | Exact duplicate quote rows (simulated double extract) | 35 rows | `Remove Duplicates` on QuoteID |
| FactQuote | Null `ProcessingDurationHours` on otherwise-completed rows | 120 rows | Conditional fill / flag as missing, don't impute silently |
| FactQuote | `ReceivedDateTime` in `DD/MM/YYYY HH:MM` text format instead of ISO | 60 rows | Custom date-parsing step with two format branches |
| FactQualityReview | Trailing whitespace on `ErrorCategoryID` | 25 rows | `Text.Trim` before merging with DimErrorCategory |
| FactQualityReview | Missing `ReviewerID` (unassigned/system-queued reviews) | 15 rows | Flag as "Unassigned" category rather than dropping |

Everything else (foreign keys, dates, premiums, workflow sequencing) is clean by design — the dataset is intentionally ~85–90% analysis-ready with a realistic, bounded cleaning exercise layered on top.

---

## 4. Business Questions (30+)

**Quote Volume**
1. How has monthly quote volume trended over the 24-month period?
2. Which states generate the most quote requests?
3. What's the residential vs. commercial split, and how has it shifted month over month?
4. Which brokers drive the most volume, and how concentrated is that volume (top 10% of brokers vs. the rest)?

**Premium**
5. What is total and average premium by state, property type, and broker?
6. How does premium distribution look across the defined premium ranges?
7. Which property/construction/occupancy combinations produce the highest average premium?
8. How does claims history affect average premium?

**Quality**
9. What is the overall first-time-right percentage, and how has it trended?
10. Which error categories occur most frequently, and which carry the highest severity?
11. Which employees have the highest and lowest error rates, controlling for volume?
12. How much rework time is being lost per month, and which error categories drive it?
13. Do higher-workload periods correlate with higher error rates?

**Underwriting & Referral**
14. What percentage of quotes are underwritten in India vs. referred to Australia?
15. What are the top referral reasons, and do they cluster by state or property type?
16. How does referral rate change across premium ranges and risk-score bands?
17. What is the average underwriting turnaround time, and how does it compare for referred vs. non-referred cases?

**Binding & Conversion**
18. What is the overall quote-to-bind conversion rate, and how does it vary by broker, state, and premium range?
19. What are the most common lost-quote reasons?
20. How much quoted premium is being lost to non-conversion each month (lost premium opportunity)?
21. Which brokers convert best relative to their volume?

**Operational Performance**
22. What is the average and median turnaround time at each workflow stage (rater, QA, underwriting, referral, QSTB)?
23. Where is the biggest bottleneck in the quote lifecycle?
24. What percentage of quotes breach a defined SLA target at each stage?
25. How does employee productivity (quotes processed) compare across teams and experience levels?

**Broker & Employee Performance Frameworks**
26. Which brokers combine high volume, high conversion, and low error rate (the "ideal" broker profile)?
27. Which employees show high volume but disproportionately high error rate (coaching candidates)?
28. Which employees are fastest without sacrificing accuracy?
29. How does tenure (time since joining) correlate with error rate and speed?

**Cross-Cutting / Advanced**
30. What does the full funnel conversion look like stage by stage (Request → Rater → QA → Underwriting → QSTB → Bound), and where is the largest drop-off?
31. Is there a relationship between quote complexity (commercial, high sum insured, multiple properties) and processing time?
32. How seasonal is the business, and does seasonality differ by state or broker type?
33. Which error categories have the strongest relationship with downstream referral or lost-quote outcomes?

---

## 5. Suggested Report Structure (refined from your original 8-page plan)

1. **Executive Overview** — Quote Requests, Total Quoted Premium, Bound Policies, Binding Rate, Avg Premium, SLA Compliance; quote trend, quote-to-bind funnel, state map, residential vs. commercial split.
2. **Quote Operations** — volume by broker/state/property type/employee, monthly trend, rater/duplicate/abandoned breakdown.
3. **Quality Dashboard** — error count/rate trend, error category and severity breakdown, FTR%, rework time, employee quality ranking.
4. **Underwriting & Referral** — premium and risk distribution, India vs. Australia split, referral reasons, referral turnaround.
5. **Broker Performance** — volume, premium, conversion, SLA impact, ranked broker scorecard (drillthrough per broker).
6. **Employee Performance** — a balanced scorecard combining volume, accuracy, and speed (avoid ranking on volume alone) — good candidate for a quadrant chart (volume vs. error rate) and a Top-N field parameter.
7. **Binding & Conversion** — funnel visual, lost-reason breakdown, lost-premium-opportunity trend, broker conversion ranking.
8. **Process Efficiency** — stage-by-stage turnaround with SLA target lines, bottleneck identification, referral delay impact.

**Interactivity to demonstrate:**
- **Field parameters** — let the user toggle the funnel/trend chart between Quotes, Premium, and Bound Count.
- **Dynamic titles** — title changes to reflect active slicer selection ("Binding Rate — NSW, Last 6 Months").
- **Dynamic KPI cards** — swap the KPI card measure (e.g., Total Premium ↔ Average Premium) via a field parameter or bookmark toggle.
- **Drillthrough** — from Broker Performance page to a broker-level detail page; from Employee Performance to an individual employee scorecard.
- **Report-page tooltips** — hover a state on the map to show a mini quote-volume trend for that state.
- **Bookmarks** — toggle between "All Quotes" and "Bound Only" views on the Executive Overview.
- **Conditional formatting** — SLA breach highlighting on the Process Efficiency page; error-rate heatmap on the Employee Performance table.
- **Top N / dynamic ranking** — Top 10 brokers by premium, adjustable via a what-if parameter.

---

## 6. DAX Measure Ideas

Core: Total Quotes, Total Premium, Average Premium, Bound Policies, Binding Rate, Referral Rate, Error Count, Error Rate, First Time Right %, SLA %, Average TAT, Median TAT.

Time intelligence: MoM Growth %, YoY Growth %, YTD Premium, Previous Year Premium, Premium Growth %, Rolling 3-Month Average Quotes.

Funnel/conversion: Quote→Rater %, Rater→QA %, QA→Underwriting %, Underwriting→QSTB %, QSTB→Bound %, Overall Conversion %, Lost Premium Opportunity.

Employee: Employee Productivity (Quotes ÷ Working Days), Employee Quality Score (weighted blend of FTR% and error severity), Employee Speed Index (Avg TAT vs. team average).

Advanced (to demonstrate strong DAX skills): a **Calculation Group** for a single "Selected Metric" that switches between Premium/Quotes/Bound Count across every visual without duplicating measures; a **weighted Employee Performance Score** combining normalized volume, accuracy, and speed into one comparable index; **SLA Breach %** using `CALCULATE` with stage-specific turnaround thresholds; **Lost Premium Opportunity** using `CALCULATE(SUM(Premium), FactBinding[BoundFlag] = FALSE)`.

---

## 7. Advanced Insights Beyond the Original Brief

- **Broker quality vs. volume trade-off** — a scatter of broker volume against their submission error rate can surface high-volume brokers who are quietly costing more rework time than their business is worth.
- **Risk-adjusted broker profitability proxy** — combine average bound premium with referral rate per broker; brokers who generate large premiums but also drive disproportionate referral/complexity work differently from brokers who convert cleanly.
- **Seasonality-adjusted SLA performance** — SLA breaches likely cluster in the higher-volume months; splitting SLA % by season isolates whether breaches are a capacity problem or a consistent process gap.
- **Employee ramp curve** — plotting error rate against tenure (joining date to current) for each employee can show how long it typically takes a new agent to reach team-average accuracy — useful for training investment conversations.
- **Referral cost in turnaround days** — quantify exactly how many extra days a referral adds to the funnel on average, and whether that delay itself correlates with a lower bind rate (does making the broker wait cost you the deal?).

---

## 8. Portfolio Story

> "I built an end-to-end insurance operations analytics solution in Power BI that models the complete quote-to-bind lifecycle for a property insurance MGA — from broker submission through quality review, underwriting (including the India/Australia referral split), quote delivery, and binding. Using a synthetic dataset built to reflect realistic operational relationships (rather than random data), the model tracks quoting productivity, first-time-right quality, underwriting referral drivers, stage-by-stage SLA performance, and broker and employee conversion patterns. The report uses a proper star schema with role-playing dimensions, DAX time intelligence, a balanced employee performance framework, and interactive features like field parameters, drillthrough, and dynamic KPI cards — the same techniques I'd apply to a real operational reporting problem, built on data I designed myself since the actual production data is confidential."

