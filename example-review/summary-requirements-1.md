# Summary: Requirements Review
**Phase:** requirements | **Iteration:** 1 | **Date:** 2026-04-08

## Finding Counts

| Reviewer | CRT | MAJ | MIN | QST | Total |
|---|---|---|---|---|---|
| PO  | 3 | 8 | 3 | 4 | 18 |
| PE  | 3 | 7 | 4 | 3 | 17 |
| UXE | 2 | 6 | 4 | 2 | 14 |
| EE  | 5 | 15 | 8 | 5 | 33 |
| UXQA | 3 | 5 | 5 | 3 | 16 |
| EQA | 4 | 15 | 10 | 4 | 33 |
| FDR | 5 | 13 | 8 | 4 | 30 |
| DBE | 3 | 5 | 5 | 3 | 16 |
| **Total** | **28** | **74** | **47** | **28** | **177** |

## Consolidated Findings

### CRT

| ID | Citation | Issue | Recommendation |
|---|---|---|---|
| META-no-ac-CRT-01 | [PO-no-ac-CRT-01](product-owner-requirements-1.md#PO-no-ac-CRT-01) | Every requirement across all files is missing the mandatory Acceptance Criteria column. The requirements principles state: "A requirement without acceptance criteria is a wish." The spec table format requires four columns (Path, Tag, Requirement, Acceptance Criteria) but all files use only three. No requirement in the MVP can be declared satisfied by an external observer. | Add an Acceptance Criteria column to every requirements table and populate it for each row with observable, testable conditions. |
| | [PE-no-ac-CRT-01](principal-engineer-requirements-1.md#PE-no-ac-CRT-01) | | |
| | [UXE-no-ac-CRT-01](ux-engineer-requirements-1.md#UXE-no-ac-CRT-01) | | |
| | [EE-no-ac-CRT-01](engine-engineer-requirements-1.md#EE-no-ac-CRT-01) | | |
| | [UXQA-no-ac-CRT-03](ux-qa-engineer-requirements-1.md#UXQA-no-ac-CRT-03) | | |
| | [EQA-no-ac-CRT-01](engine-qa-engineer-requirements-1.md#EQA-no-ac-CRT-01) | | |
| | [DBE-no-acceptance-MAJ-05](db-engineer-requirements-1.md#DBE-no-acceptance-MAJ-05) | | |
| META-surplus-routing-CRT-02 | [EE-surplus-routing-CRT-02](engine-engineer-requirements-1.md#EE-surplus-routing-CRT-02) | Surplus/deficit is computed but nothing specifies what happens with it. No requirement connects a surplus to account deposits or a deficit to the withdrawal priority sequence. This is the central cash-flow loop of the projection and it is incomplete — the balance recurrence has no net-flow input and the projection cannot close. | Add requirements specifying: (a) surplus is deposited into a designated account, (b) deficit triggers withdrawals per the configured withdrawal priority sequence. |
| | [PO-surplus-deficit-unresolved-MAJ-03](product-owner-requirements-1.md#PO-surplus-deficit-unresolved-MAJ-03) | | |
| | [PE-deficit-flow-MAJ-02](principal-engineer-requirements-1.md#PE-deficit-flow-MAJ-02) | | |
| | [EQA-surplus-routing-MAJ-09](engine-qa-engineer-requirements-1.md#EQA-surplus-routing-MAJ-09) | | |
| | [FDR-surplus-routing-QST-04](financial-domain-reviewer-requirements-1.md#FDR-surplus-routing-QST-04) | | |
| META-ss-claiming-CRT-03 | [FDR-ss-claim-range-CRT-01](financial-domain-reviewer-requirements-1.md#FDR-ss-claim-range-CRT-01) | SS claiming age constraint references FRA as a range, but FRA is a single age. The actual SSA claiming window is 62-70. Additionally, no early-claiming reduction rate/formula is specified — only delayed credits (post-FRA) are referenced. SSA applies a different formula for early claiming (5/9 of 1% per month for first 36 months, 5/12 of 1% beyond). Without both adjustments, any member claiming before FRA gets an incorrect benefit. | Restate claiming range as 62-70. Add an early-claiming reduction parameter alongside the existing delayed-credit-rate. Ensure the claiming-adjustment requirement references both early and delayed adjustments. |
| | [FDR-ss-early-reduction-CRT-02](financial-domain-reviewer-requirements-1.md#FDR-ss-early-reduction-CRT-02) | | |
| | [PO-ss-claiming-range-MAJ-08](product-owner-requirements-1.md#PO-ss-claiming-range-MAJ-08) | | |
| | [PE-ss-range-MAJ-04](principal-engineer-requirements-1.md#PE-ss-range-MAJ-04) | | |
| | [PE-early-ss-MAJ-03](principal-engineer-requirements-1.md#PE-early-ss-MAJ-03) | | |
| | [EE-ss-early-claim-MAJ-06](engine-engineer-requirements-1.md#EE-ss-early-claim-MAJ-06) | | |
| | [EE-ss-early-reduce-MAJ-07](engine-engineer-requirements-1.md#EE-ss-early-reduce-MAJ-07) | | |
| | [EQA-ss-claiming-range-MAJ-02](engine-qa-engineer-requirements-1.md#EQA-ss-claiming-range-MAJ-02) | | |
| | [EQA-ss-early-claiming-MAJ-01](engine-qa-engineer-requirements-1.md#EQA-ss-early-claiming-MAJ-01) | | |
| META-health-phases-CRT-04 | [FDR-health-per-member-CRT-05](financial-domain-reviewer-requirements-1.md#FDR-health-per-member-CRT-05) | Health insurance phases (employer, ACA, Medicare) use "at least one household member" logic, which is incorrect for two-member households with different ages. When member A reaches Medicare at 65 and member B is 60, the model would switch the entire household to Medicare, leaving the younger spouse uninsured. Phases must be per-member with independent transitions. | Rewrite insurance phase requirements to be per-member. Each member independently transitions employer -> ACA -> Medicare. Household expense is the sum of individual insurance costs. |
| | [FDR-employer-ins-phase-MAJ-09](financial-domain-reviewer-requirements-1.md#FDR-employer-ins-phase-MAJ-09) | | |
| | [EQA-employer-ins-phase-MAJ-14](engine-qa-engineer-requirements-1.md#EQA-employer-ins-phase-MAJ-14) | | |
| | [EQA-aca-phase-overlap-MAJ-15](engine-qa-engineer-requirements-1.md#EQA-aca-phase-overlap-MAJ-15) | | |
| | [EE-insurance-overlap-QST-01](engine-engineer-requirements-1.md#EE-insurance-overlap-QST-01) | | |
| META-cli-input-CRT-05 | [DBE-data-mgmt-defer-CRT-03](db-engineer-requirements-1.md#DBE-data-mgmt-defer-CRT-03) | No requirement anywhere specifies how the user provides plan data to the CLI. The CLI spec defines output (projection table, JSON) but not input. Persistence and data management are both deferred, creating a gap where the MVP has no defined mechanism for plan data entry, storage, or recovery. | Add a minimal MVP requirement for plan input: file path argument, what happens when the file is missing/malformed, and what constitutes a minimally valid plan. |
| | [PO-no-input-workflow-MAJ-02](product-owner-requirements-1.md#PO-no-input-workflow-MAJ-02) | | |
| | [UXE-no-input-spec-MAJ-05](ux-engineer-requirements-1.md#UXE-no-input-spec-MAJ-05) | | |
| | [PE-cli-input-QST-03](principal-engineer-requirements-1.md#PE-cli-input-QST-03) | | |
| | [UXQA-input-spec-QST-01](ux-qa-engineer-requirements-1.md#UXQA-input-spec-QST-01) | | |
| META-cli-errors-CRT-06 | [UXE-no-error-feedback-CRT-02](ux-engineer-requirements-1.md#UXE-no-error-feedback-CRT-02) | CLI validation requirements (missing assumptions, allocation sum) say the projection "must not run" but specify no user-visible error behavior. A silent exit, opaque error, or crash all satisfy the wording. The user needs actionable feedback identifying what is wrong. | Add requirements that validation failures produce clear error messages identifying which assumptions are missing or which allocations fail, with specific exit codes for machine consumers. |
| | [UXQA-err-msg-CRT-01](ux-qa-engineer-requirements-1.md#UXQA-err-msg-CRT-01) | | |
| | [UXQA-err-msg-CRT-02](ux-qa-engineer-requirements-1.md#UXQA-err-msg-CRT-02) | | |
| META-persist-CRT-07 | [DBE-persist-defer-CRT-01](db-engineer-requirements-1.md#DBE-persist-defer-CRT-01) | Persistence is entirely deferred, yet MVP depends on stored state: shipped defaults, per-year overrides, carry-forward semantics, and policy tables all presuppose stored configuration. Without a minimal persistence requirement, the MVP has no defined mechanism for supplying or recovering plan inputs between sessions. | Add narrow MVP persistence requirements: storage denomination (YZV per bedrock), round-trip guarantee, and plan file as authoritative source of truth. |
| | [DBE-valid-defer-CRT-02](db-engineer-requirements-1.md#DBE-valid-defer-CRT-02) | | |
| ~~META-denom-names-CRT-08~~ FIXED | [PE-denom-names-CRT-02](principal-engineer-requirements-1.md#PE-denom-names-CRT-02) | ~~Bedrock principles define YZV/CNV/YNV/PNV. Conceptual model defines YZV/CYV/FYV/PYV. Beyond casing, the names diverge: "Current Nominal Value" vs "Current Year Value," etc. All requirements reference the conceptual model's abbreviations. The inconsistency risks silent misinterpretation in schema fields, API contracts, and code.~~ Resolved: conceptual-model.md and denomination.md aligned to bedrock terminology (CYV→CNV, FYV→YNV, PYV→PNV). | No further action needed. |
| | [DBE-denom-naming-MIN-01](db-engineer-requirements-1.md#DBE-denom-naming-MIN-01) | | |
| ~~META-denom-convert-CRT-09~~ REJECTED | [EE-denom-convert-CRT-03](engine-engineer-requirements-1.md#EE-denom-convert-CRT-03) | ~~No requirement mandates that the engine can convert between denomination frames.~~ Rejected: out of scope for MVP. | No action. |
| META-tax-circular-CRT-10 | [EE-tax-circular-CRT-04](engine-engineer-requirements-1.md#EE-tax-circular-CRT-04) | Tax computation depends on withdrawals (taxable), withdrawals depend on deficit (income minus expenses minus tax), creating a circular dependency. No requirement specifies an ordering or resolution strategy. | Add a requirement specifying computation ordering that resolves the tax-withdrawal circularity. |
| ~~META-precision-CRT-11~~ FIXED | [EQA-no-precision-CRT-02](engine-qa-engineer-requirements-1.md#EQA-no-precision-CRT-02) | ~~No numerical precision, rounding mode, or cumulative tolerance requirement exists anywhere.~~ Resolved: added `R:engine / core / precision` (arbitrary-precision decimal) and `R:engine / core / rounding` (rounding is a presentation-layer concern) in [core.md](../requirements/engine/core.md). | No further action needed. |
| ~~META-conservation-CRT-12~~ REJECTED | [EQA-no-conservation-CRT-03](engine-qa-engineer-requirements-1.md#EQA-no-conservation-CRT-03) | ~~No conservation-of-assets invariant.~~ Rejected: out of scope for MVP. | No action. |
| META-rmd-tiers-CRT-13 | [FDR-rmd-age-tiers-CRT-03](financial-domain-reviewer-requirements-1.md#FDR-rmd-age-tiers-CRT-03) | The SECURE 2.0 three-tier RMD start-age mapping (born <=1950: 72, 1951-1959: 73, >=1960: 75) is not specified in requirements or conceptual model. If shipped defaults are wrong, every RMD projection starts at the wrong age. | Specify the three-tier mapping as an acceptance criterion for the RMD start-age requirement. |
| META-roth-qualified-CRT-14 | [FDR-roth-ira-5yr-CRT-04](financial-domain-reviewer-requirements-1.md#FDR-roth-ira-5yr-CRT-04) | Roth IRA "qualified withdrawal" is undefined. Under IRC 408A, qualification requires 5-year holding period AND age 59.5+. Non-qualified withdrawals have different tax treatment. For early retirees tapping Roth IRAs before 59.5, this is a material distinction. | Define qualification criteria (5-year rule, age 59.5) or document the simplifying assumption that MVP treats all Roth withdrawals as qualified. |
| | [PE-roth-ira-no-ac-reqs-MIN-03](principal-engineer-requirements-1.md#PE-roth-ira-no-ac-reqs-MIN-03) | | |
| | [EQA-roth-ira-qualified-MIN-07](engine-qa-engineer-requirements-1.md#EQA-roth-ira-qualified-MIN-07) | | |
| META-rmd-balance-CRT-15 | [EQA-rmd-balance-timing-CRT-04](engine-qa-engineer-requirements-1.md#EQA-rmd-balance-timing-CRT-04) | RMD computation does not specify which balance is used as the numerator. IRS requires prior year-end balance (Dec 31 of Y-1). Without this, the RMD amount is ambiguous and tests cannot verify correctness. | Specify that RMD for year Y uses the account balance as of Dec 31 of Y-1 divided by the applicable divisor. |
| | [EE-rmd-balance-MAJ-05](engine-engineer-requirements-1.md#EE-rmd-balance-MAJ-05) | | |
| META-broken-links-CRT-16 | [PE-broken-links-CRT-03](principal-engineer-requirements-1.md#PE-broken-links-CRT-03) | `conceptual-model.md` does not exist on disk at `requirements/conceptual-model.md`. All 92 relative links across 12 requirements files are broken for human navigation and tools. The file is delivered only via the `@` directive in CLAUDE.md. | Create `requirements/conceptual-model.md` on disk (or a symlink) so relative links resolve. |

### MAJ

| ID | Citation | Issue | Recommendation |
|---|---|---|---|
| META-account-gaps-MAJ-01 | [PO-ira-gap-CRT-03](product-owner-requirements-1.md#PO-ira-gap-CRT-03) | Multiple account types defined in the conceptual model have no MVP requirements: traditional IRA, brokerage, Roth 401k, and HSA. Traditional IRA and brokerage are among the most common retirement accounts. The conceptual model and assumptions reference these types (IRA limits, deductibility phase-outs, capital gains brackets, HSA limits) but no engine requirement file covers their tax treatment, contributions, withdrawals, or growth. | For each missing account type, either add MVP requirements or explicitly tag FUT with scope documentation. At minimum, traditional IRA should be MVP since the IRA limit enforcement in assumptions.md requires it. |
| | [PO-brokerage-gap-CRT-02](product-owner-requirements-1.md#PO-brokerage-gap-CRT-02) | | |
| | [PO-roth-401k-gap-MAJ-05](product-owner-requirements-1.md#PO-roth-401k-gap-MAJ-05) | | |
| | [PO-hsa-gap-MAJ-06](product-owner-requirements-1.md#PO-hsa-gap-MAJ-06) | | |
| | [EE-ira-trad-missing-MAJ-13](engine-engineer-requirements-1.md#EE-ira-trad-missing-MAJ-13) | | |
| | [EE-brokerage-missing-MAJ-12](engine-engineer-requirements-1.md#EE-brokerage-missing-MAJ-12) | | |
| | [EE-hsa-missing-MAJ-11](engine-engineer-requirements-1.md#EE-hsa-missing-MAJ-11) | | |
| | [EE-401k-roth-missing-MIN-01](engine-engineer-requirements-1.md#EE-401k-roth-missing-MIN-01) | | |
| | [EQA-ira-missing-MAJ-06](engine-qa-engineer-requirements-1.md#EQA-ira-missing-MAJ-06) | | |
| | [EQA-brokerage-missing-MAJ-07](engine-qa-engineer-requirements-1.md#EQA-brokerage-missing-MAJ-07) | | |
| | [EQA-hsa-missing-MAJ-05](engine-qa-engineer-requirements-1.md#EQA-hsa-missing-MAJ-05) | | |
| | [EQA-roth-401k-missing-MAJ-08](engine-qa-engineer-requirements-1.md#EQA-roth-401k-missing-MAJ-08) | | |
| | [FDR-ira-missing-MAJ-03](financial-domain-reviewer-requirements-1.md#FDR-ira-missing-MAJ-03) | | |
| | [FDR-brokerage-missing-MAJ-04](financial-domain-reviewer-requirements-1.md#FDR-brokerage-missing-MAJ-04) | | |
| | [FDR-roth-401k-missing-MAJ-01](financial-domain-reviewer-requirements-1.md#FDR-roth-401k-missing-MAJ-01) | | |
| META-catchup-MAJ-02 | [EE-catchup-401k-MAJ-09](engine-engineer-requirements-1.md#EE-catchup-401k-MAJ-09) | Catchup contributions (401k and IRA) are defined in the conceptual model and assumptions but have no contribution requirements. The assumptions define catchup and super-catchup limits with SECURE 2.0 age bands (50-59/64+ catchup, 60-63 super-catchup), but contributions.md contains no requirement for catchup contributions. Users over 50 can contribute significantly more. | Add explicit catchup contribution requirements for 401k and Roth IRA, including age-band eligibility and applicable limits, or tag as FUT. |
| | [FDR-401k-catchup-ages-MAJ-06](financial-domain-reviewer-requirements-1.md#FDR-401k-catchup-ages-MAJ-06) | | |
| | [EQA-401k-catchup-MAJ-03](engine-qa-engineer-requirements-1.md#EQA-401k-catchup-MAJ-03) | | |
| | [EE-roth-ira-catchup-MAJ-10](engine-engineer-requirements-1.md#EE-roth-ira-catchup-MAJ-10) | | |
| | [EQA-roth-ira-catchup-MAJ-04](engine-qa-engineer-requirements-1.md#EQA-roth-ira-catchup-MAJ-04) | | |
| | [PO-catchup-contrib-MIN-03](product-owner-requirements-1.md#PO-catchup-contrib-MIN-03) | | |
| | [PE-catchup-gap-MIN-02](principal-engineer-requirements-1.md#PE-catchup-gap-MIN-02) | | |
| META-filing-status-MAJ-03 | [FDR-tax-filing-status-MAJ-08](financial-domain-reviewer-requirements-1.md#FDR-tax-filing-status-MAJ-08) | Filing status is implied by multiple requirements (tax brackets, standard deduction, SS taxation thresholds, Roth IRA phase-outs, NIIT thresholds) but never explicitly required or defined. No requirement specifies how filing status is determined from household composition or whether it is configurable. | Add a requirement for filing status as a household attribute, with derivation rules (e.g., MFJ when spouse exists, single otherwise) and specification of which downstream computations depend on it. |
| | [PO-filing-status-QST-02](product-owner-requirements-1.md#PO-filing-status-QST-02) | | |
| | [PE-two-member-tax-QST-01](principal-engineer-requirements-1.md#PE-two-member-tax-QST-01) | | |
| | [EE-filing-status-QST-02](engine-engineer-requirements-1.md#EE-filing-status-QST-02) | | |
| | [EQA-two-earner-tax-QST-04](engine-qa-engineer-requirements-1.md#EQA-two-earner-tax-QST-04) | | |
| META-alloc-scope-MAJ-04 | [EE-alloc-per-acct-MAJ-03](engine-engineer-requirements-1.md#EE-alloc-per-acct-MAJ-03) | Allocation scope is unclear: the conceptual model supports per-account overrides, but requirements only mention a single configurable allocation. The rate-derivation requirement says "weighted average of rates using allocations as weights" without specifying whether this is global or per-account. In practice, most users have different allocations per account. | Clarify whether MVP uses a single plan-wide allocation or per-account overrides. State explicitly which is in scope. |
| | [PO-allocation-per-account-QST-01](product-owner-requirements-1.md#PO-allocation-per-account-QST-01) | | |
| | [PE-alloc-scope-MIN-01](principal-engineer-requirements-1.md#PE-alloc-scope-MIN-01) | | |
| | [EQA-rate-derivation-allocation-MAJ-11](engine-qa-engineer-requirements-1.md#EQA-rate-derivation-allocation-MAJ-11) | | |
| | [FDR-alloc-per-acct-MIN-08](financial-domain-reviewer-requirements-1.md#FDR-alloc-per-acct-MIN-08) | | |
| META-taxable-income-MAJ-05 | [PE-taxable-income-inputs-MAJ-07](principal-engineer-requirements-1.md#PE-taxable-income-inputs-MAJ-07) | No requirement enumerates which income streams compose taxable income. W-2, SS benefits (partially taxable), 401k withdrawals, and other sources have different tax treatments, but there is no aggregation requirement tying them together. Without this, the engine cannot compute taxable income correctly. | Add a requirement for gross income composition that explicitly lists which income and withdrawal streams contribute, their inclusion rules, and the order of deductions. |
| | [EE-expense-tax-MAJ-14](engine-engineer-requirements-1.md#EE-expense-tax-MAJ-14) | | |
| | [EQA-tax-which-income-MAJ-10](engine-qa-engineer-requirements-1.md#EQA-tax-which-income-MAJ-10) | | |
| META-plan-end-MAJ-06 | [PO-member-death-year-MAJ-07](product-owner-requirements-1.md#PO-member-death-year-MAJ-07) | No requirement defines a plan end condition, member death year, or life expectancy. The conceptual model says each member's stream spans "birth year to death year" but no requirement makes death year configurable. Without this, the projection range is undefined and design cannot determine when the timeline ends. | Add a requirement for configurable life expectancy or plan end year per member, and specify behavior for the surviving spouse. |
| | [PE-no-plan-end-MAJ-01](principal-engineer-requirements-1.md#PE-no-plan-end-MAJ-01) | | |
| META-roth-conv-MAJ-07 | [EE-roth-conv-CRT-05](engine-engineer-requirements-1.md#EE-roth-conv-CRT-05) | Roth conversions are defined in the conceptual model but absent from all engine requirements. Roth conversions have significant tax implications (converted amount is taxable income) and are one of the most impactful retirement planning strategies. | Add Roth conversion requirements (taxable amount, source/target accounts, per-year configurability) or tag FUT explicitly. |
| | [FDR-roth-conv-missing-MAJ-02](financial-domain-reviewer-requirements-1.md#FDR-roth-conv-missing-MAJ-02) | | |
| | [EQA-roth-conversion-missing-MIN-08](engine-qa-engineer-requirements-1.md#EQA-roth-conversion-missing-MIN-08) | | |
| META-w2-growth-MAJ-08 | [PO-income-growth-QST-03](product-owner-requirements-1.md#PO-income-growth-QST-03) | W-2 income growth/inflation is unspecified. For a multi-decade projection, flat nominal salary produces very different results than one growing at CPI or a configured raise rate. No requirement specifies denomination of the configured amount or whether it inflates. | Specify whether W-2 income inflates (at CPI, a wage-growth rate, or user-configured rate) and the denomination of the configured value. |
| | [EE-w2-growth-MIN-03](engine-engineer-requirements-1.md#EE-w2-growth-MIN-03) | | |
| | [EQA-w2-income-denomination-MIN-01](engine-qa-engineer-requirements-1.md#EQA-w2-income-denomination-MIN-01) | | |
| | [FDR-w2-growth-MIN-03](financial-domain-reviewer-requirements-1.md#FDR-w2-growth-MIN-03) | | |
| META-bank-growth-MAJ-09 | [EE-bank-growth-MAJ-04](engine-engineer-requirements-1.md#EE-bank-growth-MAJ-04) | Bank account growth rate is unspecified. The rate-derivation requirement applies an equity/bond weighted average to all accounts, but a bank account typically earns a savings rate near zero. Applying the portfolio rate to cash overstates reserves. | Add a requirement that bank account growth is independently configurable (defaulting to 0%) and exempt from the allocation-weighted rate. |
| | [PO-bank-growth-MIN-01](product-owner-requirements-1.md#PO-bank-growth-MIN-01) | | |
| | [EQA-bank-growth-MIN-05](engine-qa-engineer-requirements-1.md#EQA-bank-growth-MIN-05) | | |
| META-ss-taxation-MAJ-10 | [FDR-ss-taxation-formula-MAJ-05](financial-domain-reviewer-requirements-1.md#FDR-ss-taxation-formula-MAJ-05) | SS taxation requirement references thresholds but does not specify the provisional income formula (AGI excluding SS + 50% of SS benefits) or the two-tier structure (0%/50%/85% taxable). Without the formula, the thresholds alone are insufficient to determine the taxable portion. | Add a requirement for provisional income computation and the two-tier taxation structure. |
| | [EQA-ss-taxation-formula-QST-02](engine-qa-engineer-requirements-1.md#EQA-ss-taxation-formula-QST-02) | | |
| META-json-MAJ-11 | [UXE-json-unspecified-MAJ-06](ux-engineer-requirements-1.md#UXE-json-unspecified-MAJ-06) | JSON output is required but its content, schema, and error behavior are all unspecified. No requirement says what the JSON must contain, whether it mirrors the table columns, or how errors are reported in JSON mode. Test infrastructure cannot reliably consume it. | Specify JSON output content (at minimum the same data as the table), and require structured error output in JSON mode. |
| | [UXQA-json-schema-MAJ-03](ux-qa-engineer-requirements-1.md#UXQA-json-schema-MAJ-03) | | |
| | [UXQA-json-err-MAJ-02](ux-qa-engineer-requirements-1.md#UXQA-json-err-MAJ-02) | | |
| META-first-run-MAJ-12 | [PO-no-first-run-MAJ-01](product-owner-requirements-1.md#PO-no-first-run-MAJ-01) | No plan creation workflow or first-run experience is specified. The tool has no defined entry point — no requirement describes minimum inputs for a first projection or what the user does when they first launch the tool. | Add a requirement for minimum user inputs needed for a first projection and that all other values use shipped defaults. |
| | [DBE-plan-creation-QST-02](db-engineer-requirements-1.md#DBE-plan-creation-QST-02) | | |
| META-magi-MAJ-13 | [PE-magi-MAJ-05](principal-engineer-requirements-1.md#PE-magi-MAJ-05) | MAGI (Modified Adjusted Gross Income) is referenced in the Roth IRA phase-out requirement but never defined. MAGI differs from taxable income and AGI. Without a computation requirement, the phase-out cannot be evaluated. | Add a requirement for MAGI computation specifying which income sources contribute. |
| META-net-flow-MAJ-14 | [EE-net-flow-MAJ-01](engine-engineer-requirements-1.md#EE-net-flow-MAJ-01) | The "net-flow" term in the balance recurrence formula `p(y) = p(y-1) * (1 + rate) + net-flow` is not defined anywhere. It presumably equals contributions minus withdrawals but this is never stated. | Add a requirement defining net-flow for each account type as the sum of all inflows minus all outflows for that account in that year. |
| META-pia-MAJ-15 | [EE-pia-input-MAJ-08](engine-engineer-requirements-1.md#EE-pia-input-MAJ-08) | PIA (Primary Insurance Amount) is the foundation of SS benefit calculation but its source (user input vs computed from earnings) and denomination are unspecified. | Specify that PIA is a user-configured input per member with a defined denomination and reference year for COLA application. |
| | [EQA-ss-pia-denomination-MIN-02](engine-qa-engineer-requirements-1.md#EQA-ss-pia-denomination-MIN-02) | | |
| META-floor-behavior-MAJ-16 | [EQA-floor-enforcement-MAJ-12](engine-qa-engineer-requirements-1.md#EQA-floor-enforcement-MAJ-12) | Account balance floor ("no balance below zero") does not specify enforcement behavior. When a withdrawal would go negative: is it truncated? Does the remainder cascade to the next account? Is the shortfall an error? | Specify: cap withdrawal at available balance, cascade unfunded remainder to next account in withdrawal priority. |
| | [EE-floor-behavior-MIN-06](engine-engineer-requirements-1.md#EE-floor-behavior-MIN-06) | | |
| META-rmd-exempt-MAJ-17 | [FDR-rmd-roth-exempt-MAJ-07](financial-domain-reviewer-requirements-1.md#FDR-rmd-roth-exempt-MAJ-07) | RMD requirements exist only for 401k. Traditional IRA RMDs are missing (needed if IRA is MVP). Roth IRA is exempt from RMDs (not stated). Roth 401k was exempt starting 2024 (not stated). | Add IRA RMDs, add explicit Roth IRA RMD exemption, clarify Roth 401k RMD status per SECURE 2.0. |
| | [EE-rmd-ira-QST-03](engine-engineer-requirements-1.md#EE-rmd-ira-QST-03) | | |
| META-income-earning-MAJ-18 | [PE-income-earning-MAJ-06](principal-engineer-requirements-1.md#PE-income-earning-MAJ-06) | The term "income-earning" qualifies household member in six requirements but is never defined. A dependent could earn income; a spouse might not. The boundary is ambiguous. | Define "income-earning" as a stream attribute or member classification with clear criteria. |
| META-carry-forward-MAJ-19 | [DBE-carry-forward-MAJ-04](db-engineer-requirements-1.md#DBE-carry-forward-MAJ-04) | Carry-forward semantics are required but behavior at boundaries is undefined: what value applies before the first override? What happens when an override is removed? | Define carry-forward: value set in year Y applies to Y, Y+1, ... until next override. Shipped default is the initial anchor. |
| | [EQA-carry-forward-semantics-MIN-04](engine-qa-engineer-requirements-1.md#EQA-carry-forward-semantics-MIN-04) | | |
| META-rate-scope-MAJ-20 | [EE-rate-scope-MAJ-02](engine-engineer-requirements-1.md#EE-rate-scope-MAJ-02) | Only 2 rate streams are MVP (large-cap, bonds) but the allocation model references 4 asset classes. Mismatch between available rate streams and allocation categories. | Ensure the number of MVP rate streams matches the MVP allocation categories, or restrict MVP allocations to the two available classes. |
| META-policy-data-MAJ-21 | [DBE-policy-data-MAJ-02](db-engineer-requirements-1.md#DBE-policy-data-MAJ-02) | Policy data tables (tax brackets, limits, RMD tables) have no versioning, sourcing, or update lifecycle requirement. The user cannot tell what vintage of policy data they are running against. | Add a requirement for policy data versioning and observability. |
| | [DBE-defaults-source-MAJ-03](db-engineer-requirements-1.md#DBE-defaults-source-MAJ-03) | | |
| | [DBE-storage-denom-MAJ-01](db-engineer-requirements-1.md#DBE-storage-denom-MAJ-01) | | |
| META-net-worth-MAJ-22 | [PO-no-net-worth-MAJ-04](product-owner-requirements-1.md#PO-no-net-worth-MAJ-04) | Net worth is not defined as an engine requirement or included in the CLI output, but it is a primary orientation metric for retirement planning. | Add an engine requirement for net worth computation (sum of account balances for MVP) and include it in the CLI projection table. |
| META-rmd-floor-MAJ-23 | [EQA-rmd-floor-interaction-MAJ-13](engine-qa-engineer-requirements-1.md#EQA-rmd-floor-interaction-MAJ-13) | RMD floor interaction with voluntary withdrawals is ambiguous. "Floor on the total withdrawal" could mean RMD is additive or that voluntary withdrawals >= RMD satisfy the requirement. IRS rules say the latter. | Clarify: if total voluntary withdrawal meets or exceeds RMD, no additional withdrawal needed. Otherwise, increase to the RMD amount. |
| META-denom-label-MAJ-24 | [UXQA-denom-label-MAJ-01](ux-qa-engineer-requirements-1.md#UXQA-denom-label-MAJ-01) | CLI output does not label which denomination mode is active. Users have no way to know if values are in CYV, YZV, or FYV. | Add a requirement that CLI output indicates the active denomination reference frame. |
| META-exit-code-MAJ-25 | [UXQA-exit-code-MAJ-05](ux-qa-engineer-requirements-1.md#UXQA-exit-code-MAJ-05) | No CLI exit code requirement. Test infrastructure consuming JSON output needs distinct exit codes for success vs each failure class. | Add exit code requirements for success and each failure category. |
| META-empty-plan-MAJ-26 | [UXQA-empty-plan-MAJ-04](ux-qa-engineer-requirements-1.md#UXQA-empty-plan-MAJ-04) | No requirement for CLI behavior when plan has zero accounts, zero income, or zero expenses. | Add a requirement specifying behavior for empty/minimal plans. |
| META-contribs-in-cli-MAJ-27 | [UXE-no-contribs-withdrawals-MAJ-03](ux-engineer-requirements-1.md#UXE-no-contribs-withdrawals-MAJ-03) | CLI projection table omits contributions and withdrawals from required columns. These are core streams essential for understanding how balances change year over year. | Add contributions and withdrawals to the required CLI table columns, or document why excluded. |
| | [UXQA-contrib-visibility-MIN-04](ux-qa-engineer-requirements-1.md#UXQA-contrib-visibility-MIN-04) | | |
| | [UXQA-rmd-visibility-MIN-03](ux-qa-engineer-requirements-1.md#UXQA-rmd-visibility-MIN-03) | | |
| META-match-ceiling-MAJ-28 | [EE-employer-match-inflate-MIN-08](engine-engineer-requirements-1.md#EE-employer-match-inflate-MIN-08) | Employer match "deferral ceiling" is ambiguous — could be a percentage of salary or a dollar amount. The denomination and inflation behavior are also unspecified. | Clarify whether the ceiling is a percentage or dollar amount. If dollar, specify denomination. |
| | [EQA-401k-match-ceiling-MIN-06](engine-qa-engineer-requirements-1.md#EQA-401k-match-ceiling-MIN-06) | | |

### MIN

| ID | Citation | Issue | Recommendation |
|---|---|---|---|
| META-dependent-budget-MIN-01 | [PO-dependent-expenses-MIN-02](product-owner-requirements-1.md#PO-dependent-expenses-MIN-02) | Dependent age-out removes members from budget but no dependent-specific expenses exist in MVP. The mechanism has no observable effect. | Either add per-dependent expense streams or defer age-out to FUT. |
| | [EQA-dependent-budget-impact-MIN-10](engine-qa-engineer-requirements-1.md#EQA-dependent-budget-impact-MIN-10) | | |
| META-housing-expense-MIN-02 | [PE-expense-housing-MIN-04](principal-engineer-requirements-1.md#PE-expense-housing-MIN-04) | Housing and discretionary expenses are in the conceptual model but not in MVP requirements. Housing is typically the largest expense. If "living" is meant to encompass housing, clarify scope. | Clarify scope of "living" expense or add housing separately. |
| | [EE-expense-disc-MIN-04](engine-engineer-requirements-1.md#EE-expense-disc-MIN-04) | | |
| META-balance-init-MIN-03 | [EE-balance-init-MIN-05](engine-engineer-requirements-1.md#EE-balance-init-MIN-05) | Balance recurrence requires an initial balance (base case) but no requirement specifies how it enters the system. | Add a requirement for configurable initial account balance with denomination. |
| META-ss-cola-base-MIN-04 | [EE-ss-cola-base-MIN-07](engine-engineer-requirements-1.md#EE-ss-cola-base-MIN-07) | COLA compounding base year is unspecified. | Specify COLA applies from claiming year, compounding annually. |
| META-member-identity-MIN-05 | [DBE-member-identity-MIN-02](db-engineer-requirements-1.md#DBE-member-identity-MIN-02) | No stable identity mechanism for household members or accounts. Positional references could corrupt data on reorder. | Add requirements for stable, unique identifiers for members and accounts. |
| | [DBE-account-identity-MIN-03](db-engineer-requirements-1.md#DBE-account-identity-MIN-03) | | |
| META-birth-year-MIN-06 | [DBE-birth-year-MIN-04](db-engineer-requirements-1.md#DBE-birth-year-MIN-04) | Birth year is needed by multiple requirements but never explicitly required as a member input. | Add an explicit requirement for birth year as a mandatory member field. |
| META-alloc-tolerance-MIN-07 | [EQA-alloc-sum-tolerance-MIN-03](engine-qa-engineer-requirements-1.md#EQA-alloc-sum-tolerance-MIN-03) | Allocation sum "must equal 100%" with no tolerance for floating-point representation. | Specify a tolerance (e.g., within 0.01 percentage points). |
| META-wide-table-MIN-08 | [UXE-wide-table-MIN-04](ux-engineer-requirements-1.md#UXE-wide-table-MIN-04) | CLI table could be very wide for multi-member, multi-account households with no readability requirement. | Add a requirement for handling wide tables. |
| | [UXQA-multi-member-MIN-02](ux-qa-engineer-requirements-1.md#UXQA-multi-member-MIN-02) | | |
| META-liabilities-MIN-09 | [EE-expense-deficit-MAJ-15](engine-engineer-requirements-1.md#EE-expense-deficit-MAJ-15) | Liabilities are in the conceptual model but not addressed in MVP requirements. No FUT tag. | Either add liability requirements or explicitly tag FUT. |
| | [EQA-liabilities-missing-MIN-09](engine-qa-engineer-requirements-1.md#EQA-liabilities-missing-MIN-09) | | |
| | [UXE-liabilities-absent-MIN-03](ux-engineer-requirements-1.md#UXE-liabilities-absent-MIN-03) | | |
| META-niit-capgains-MIN-10 | [FDR-niit-missing-MAJ-10](financial-domain-reviewer-requirements-1.md#FDR-niit-missing-MAJ-10) | NIIT (3.8%), capital gains tax, SE tax, and state tax are all defined in the conceptual model but absent from engine requirements. Materiality depends on account types in scope (brokerage requires capital gains; 1099 income requires SE tax). | Tag each as MVP or FUT, consistent with the account type decisions. |
| | [FDR-cap-gains-missing-MAJ-11](financial-domain-reviewer-requirements-1.md#FDR-cap-gains-missing-MAJ-11) | | |
| | [FDR-se-tax-missing-MAJ-12](financial-domain-reviewer-requirements-1.md#FDR-se-tax-missing-MAJ-12) | | |
| | [FDR-state-tax-missing-MAJ-13](financial-domain-reviewer-requirements-1.md#FDR-state-tax-missing-MAJ-13) | | |
| | [EE-cap-gains-QST-04](engine-engineer-requirements-1.md#EE-cap-gains-QST-04) | | |
| | [EE-state-tax-QST-05](engine-engineer-requirements-1.md#EE-state-tax-QST-05) | | |
| | [UXE-state-tax-absent-MIN-01](ux-engineer-requirements-1.md#UXE-state-tax-absent-MIN-01) | | |
| META-ss-domain-MIN-11 | [FDR-ss-spousal-MIN-06](financial-domain-reviewer-requirements-1.md#FDR-ss-spousal-MIN-06) | Several SS domain features are absent: spousal benefits, earnings test, first-year RMD deferral, 401k early withdrawal penalty. Each is a documented simplifying assumption or FUT candidate. | Document each as an explicit simplifying assumption or FUT. |
| | [FDR-ss-earnings-test-MIN-07](financial-domain-reviewer-requirements-1.md#FDR-ss-earnings-test-MIN-07) | | |
| | [FDR-rmd-first-year-MIN-05](financial-domain-reviewer-requirements-1.md#FDR-rmd-first-year-MIN-05) | | |
| | [FDR-401k-early-wd-MIN-04](financial-domain-reviewer-requirements-1.md#FDR-401k-early-wd-MIN-04) | | |
| META-aca-irmaa-MIN-12 | [FDR-aca-subsidy-missing-MIN-01](financial-domain-reviewer-requirements-1.md#FDR-aca-subsidy-missing-MIN-01) | ACA subsidy calculation and IRMAA surcharges are defined in conceptual model but absent from requirements. Both materially affect healthcare costs for early retirees and high-income retirees respectively. | Tag as MVP or FUT. If deferred, document the simplification. |
| | [FDR-irmaa-missing-MIN-02](financial-domain-reviewer-requirements-1.md#FDR-irmaa-missing-MIN-02) | | |
| META-col-order-MIN-13 | [UXQA-col-order-MIN-01](ux-qa-engineer-requirements-1.md#UXQA-col-order-MIN-01) | CLI column ordering and surplus/deficit sign convention unspecified. | Specify column order and sign convention for surplus/deficit. |
| | [UXQA-surplus-sign-MIN-05](ux-qa-engineer-requirements-1.md#UXQA-surplus-sign-MIN-05) | | |
| META-json-schema-MIN-14 | [DBE-json-output-MIN-05](db-engineer-requirements-1.md#DBE-json-output-MIN-05) | JSON output schema is a machine-readable contract but is not versioned or documented. | Add versioning or documentation requirement for JSON output. |
| META-income-source-MIN-15 | [UXE-income-by-source-MAJ-04](ux-engineer-requirements-1.md#UXE-income-by-source-MAJ-04) | "Income by source" grouping level in CLI is ambiguous (per-type vs per-instance vs per-member). | Specify grouping level for income columns. |
| META-fed-tax-scope-MIN-16 | [UXE-fed-tax-ambiguous-MAJ-02](ux-engineer-requirements-1.md#UXE-fed-tax-ambiguous-MAJ-02) | "Federal tax" in CLI columns does not clarify if it means income tax only or includes FICA/SE/NIIT. | Clarify the scope of "federal tax" in the CLI output. |

### QST

| ID | Citation | Issue | Recommendation |
|---|---|---|---|
| META-inflation-idx-QST-01 | [FDR-inflation-idx-QST-02](financial-domain-reviewer-requirements-1.md#FDR-inflation-idx-QST-02) | Should policy limits (tax brackets, contribution limits) auto-inflate with CPI using IRS rounding conventions in non-overridden years, or are they static defaults? This dramatically affects long-projection accuracy. | Decide: auto-inflate with rounding conventions, or carry-forward nominal values. If auto-inflate, specify rounding per limit type. |
| | [EQA-tax-bracket-inflation-QST-03](engine-qa-engineer-requirements-1.md#EQA-tax-bracket-inflation-QST-03) | | |
| META-denom-engine-QST-02 | [PE-denom-which-frame-QST-02](principal-engineer-requirements-1.md#PE-denom-which-frame-QST-02) | No requirement specifies which denomination the engine computes in. Bedrock says "storage is YZV" and "projections produce YNV/FYV" but these are architectural constraints not requirements. | Decide whether engine computation denomination belongs in requirements or is purely a design concern. |
| META-rmd-joint-QST-03 | [FDR-rmd-joint-table-QST-01](financial-domain-reviewer-requirements-1.md#FDR-rmd-joint-table-QST-01) | The Joint Life and Last Survivor Table (Table II) is in the conceptual model but absent from requirements. Applies when sole beneficiary is a spouse 10+ years younger. | Decide if Table II is MVP or FUT. |
| META-anchor-year-QST-04 | [DBE-anchor-year-QST-01](db-engineer-requirements-1.md#DBE-anchor-year-QST-01) | Bedrock implies a rebase operation exists but no requirement addresses it. | Clarify whether rebase is in scope and if not, state anchor year is immutable. |
| META-override-denom-QST-05 | [DBE-override-denom-QST-03](db-engineer-requirements-1.md#DBE-override-denom-QST-03) | When a user overrides a dollar-denominated assumption for a future year, what denomination do they enter it in? YZV is unintuitive for users. | Specify entry denomination for per-year overrides and conversion to storage denomination. |
| META-phase-out-formula-QST-06 | [EQA-phase-out-formula-QST-01](engine-qa-engineer-requirements-1.md#EQA-phase-out-formula-QST-01) | Roth IRA phase-out reduction formula (linear, $10 rounding, $200 minimum) is not specified. Is this a requirements or design concern? | Decide whether the phase-out formula belongs in requirements. If so, specify IRS rounding rules. |
| META-denom-select-QST-07 | [UXQA-denom-select-QST-02](ux-qa-engineer-requirements-1.md#UXQA-denom-select-QST-02) | Is there an MVP mechanism for denomination mode switching on the CLI, or is CYV the only mode? | Decide: add a CLI flag for denomination mode or state CYV-only for MVP. |
| | [UXE-denom-mode-MIN-02](ux-engineer-requirements-1.md#UXE-denom-mode-MIN-02) | | |
| META-partial-val-QST-08 | [UXQA-partial-val-QST-03](ux-qa-engineer-requirements-1.md#UXQA-partial-val-QST-03) | Only two validation gates are specified in CLI (missing assumptions, allocation sum). Are other failure modes (missing birth year, invalid ages, etc.) expected to surface? | Clarify whether CLI surfaces all engine validation failures or only the two listed. |
| META-missing-assumptions-QST-09 | [UXE-missing-assumptions-scope-QST-02](ux-engineer-requirements-1.md#UXE-missing-assumptions-scope-QST-02) | "Required assumptions" is not defined. Which assumptions are required vs optional? | Define the set of required assumptions explicitly. |
| META-hsa-coverage-QST-10 | [FDR-hsa-coverage-QST-03](financial-domain-reviewer-requirements-1.md#FDR-hsa-coverage-QST-03) | HSA contribution limits depend on HDHP coverage type (self-only vs family). No mechanism determines which applies. | If HSA is MVP, add a coverage-type requirement. |

## Priority Recommendation

### Tier 1 — Structural (fix before any content review)

1. **META-no-ac-CRT-01**: Add acceptance criteria to every requirement. This is a systemic violation of the project's own principles and blocks all testability.
2. **META-denom-names-CRT-08**: Reconcile denomination abbreviations between bedrock and conceptual model. All downstream documents reference these.
3. **META-broken-links-CRT-16**: Place `conceptual-model.md` on disk at `requirements/conceptual-model.md` so all 92 relative links resolve.

### Tier 2 — Projection loop completeness (the engine cannot produce correct results without these)

4. **META-surplus-routing-CRT-02**: Define surplus/deficit routing. This closes the cash-flow loop.
5. **META-tax-circular-CRT-10**: Specify computation ordering for the tax-withdrawal circularity.
6. **META-net-flow-MAJ-14**: Define the net-flow term in the recurrence formula.
7. **META-taxable-income-MAJ-05**: Enumerate taxable income composition.
8. **META-filing-status-MAJ-03**: Define filing status — everything tax-related depends on it.
9. **META-magi-MAJ-13**: Define MAGI computation — phase-outs depend on it.

### Tier 3 — Financial rule correctness (wrong answers if not fixed)

10. **META-ss-claiming-CRT-03**: Fix SS claiming range (62-70) and add early reduction formula.
11. **META-health-phases-CRT-04**: Make health insurance phases per-member.
12. **META-rmd-tiers-CRT-13**: Specify SECURE 2.0 three-tier RMD ages.
13. **META-rmd-balance-CRT-15**: Specify prior year-end balance for RMD.
14. **META-ss-taxation-MAJ-10**: Add provisional income formula.

### Tier 4 — Scope decisions (unresolved ambiguities that block design)

15. **META-account-gaps-MAJ-01**: Decide traditional IRA, brokerage, Roth 401k, HSA scope.
16. **META-catchup-MAJ-02**: Decide catchup contribution scope.
17. **META-alloc-scope-MAJ-04**: Decide global vs per-account allocation.
18. **META-roth-conv-MAJ-07**: Decide Roth conversion scope.
19. **META-plan-end-MAJ-06**: Define plan end condition.
20. **META-w2-growth-MAJ-08**: Decide W-2 income growth behavior.

### Tier 5 — MVP usability (the tool must have these to be usable)

21. **META-cli-input-CRT-05**: Define how plan data enters the CLI.
22. **META-cli-errors-CRT-06**: Define error messaging behavior.
23. **META-persist-CRT-07**: Add minimal persistence requirements.
24. **META-first-run-MAJ-12**: Define first-run experience.
25. ~~**META-precision-CRT-11**: Add precision/rounding requirements.~~ FIXED — see [core.md](requirements/engine/core.md).
26. ~~**META-conservation-CRT-12**: Add conservation-of-assets invariant.~~ REJECTED — out of scope for MVP.
