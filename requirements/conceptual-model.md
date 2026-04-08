# Conceptual Model

This document defines the domain entities and their relationships at the
requirements level. It does not restate [bedrock principles](../bedrock-principles.md) —
it captures their behavioral implications for the system.

## D: Denomination

| Path | Description |
|------|-------------|
| denomination | Every dollar amount is denominated in exactly one of the four reference frames defined in bedrock principles. An amount without a known denomination is meaningless. |
| denomination / yzv | Year Zero Value. Dollars denominated in the plan's anchor year. Fixed at plan creation. |
| denomination / cyv | Current Year Value. Dollars denominated in the current calendar year's purchasing power. Slides forward annually. |
| denomination / fyv | Future Year Value. Dollars denominated in a specific projection year's nominal terms. Always derived, never observed. |
| denomination / pyv | Past Year Value. Dollars denominated in a specific historical year's nominal terms. Always observed, never derived. |
| denomination / display | User-facing dollar values default to current-year purchasing power (CYV). |

## D: Streams

| Path | Description |
|------|-------------|
| streams | Every time-varying quantity is a stream that produces a value for each year it is active and nothing for years it is not. |
| streams / attributes | A stream may carry attributes — concrete properties that vary over time. A stream may have multiple attributes. |
| streams / composition | A stream may compose child streams. Child streams may themselves carry attributes and compose further children. |
| streams / timeline | The plan timeline spans from the earliest stream start to the latest stream end. It is not independently configured. |

## D: Formulas

| Path | Description |
|------|-------------|
| formulas / balance-recurrence | Family of recurrence relations for projecting account balances year over year. All variants compute p(y) from p(y-1), a growth rate, and net cash flow for the year. |
| formulas / balance-recurrence / end-of-year | End-of-year convention. Growth applies to the prior balance; net flow is added after growth. p(y) = p(y-1) * (1 + rate) + net-flow. |
| formulas / balance-recurrence / beginning-of-year | Beginning-of-year convention. Net flow is added first; growth applies to the adjusted balance. p(y) = (p(y-1) + net-flow) * (1 + rate). |
| formulas / balance-recurrence / mid-year | Mid-year convention. Net flow is assumed to occur at mid-year and earns a half-year of growth. p(y) = p(y-1) * (1 + rate) + net-flow * (1 + rate/2). |

## S: Assumptions

| Path | Description |
|------|-------------|
| assumptions | Configurable parameters that drive projections. |
| assumptions / inflation | Inflation rate assumptions used for denomination conversions and value adjustments. |
| assumptions / inflation / cpi | Consumer Price Index — general inflation rate. |
| assumptions / inflation / cpi-w | CPI for Urban Wage Earners — used for Social Security COLA adjustments. |
| assumptions / inflation / health | Healthcare cost inflation rate. |
| assumptions / rates | Expected return rate assumptions by asset class. |
| assumptions / rates / large-cap | Return rate stream for large-cap equities. |
| assumptions / rates / small-cap | Return rate stream for small-cap equities. |
| assumptions / rates / international | Return rate stream for international equities. |
| assumptions / rates / bonds | Return rate stream for bonds. |
| assumptions / rates / real-estate | Return rate stream for real estate. |
| assumptions / rates / [ custom ] | A user-defined return rate stream for an additional asset class. |
| assumptions / allocations | Target asset allocation percentages across asset classes (large-cap, small-cap, international, bonds). Allocations sum to 100%. |
| assumptions / allocations / [ member ] / [ account ] | Allocation override stream for a specific member-owned account. |
| assumptions / allocations / [ account ] | Allocation override stream for a joint or household account. |
| assumptions / policy | Externally determined regulatory and legislative parameters. Values change as law changes. User may override for future years to model legislative scenarios. |
| assumptions / policy / limits | IRS annual contribution limits by account type. |
| assumptions / policy / limits / 401k | Contribution limits for traditional and Roth 401k accounts. |
| assumptions / policy / limits / 401k : max | Annual employee elective deferral limit. |
| assumptions / policy / limits / 401k : catchup | Additional elective deferral limit for members age 50–59 and 64+. |
| assumptions / policy / limits / 401k : super-catchup | Additional elective deferral limit for members age 60–63 (SECURE 2.0). |
| assumptions / policy / limits / 401k : annual-additions | Overall annual additions limit including employee and employer contributions combined. |
| assumptions / policy / limits / roth-401k | Contribution limits for Roth 401k accounts. |
| assumptions / policy / limits / roth-401k : max | Annual employee elective deferral limit. Shared with traditional 401k — combined employee contributions across both may not exceed this limit. |
| assumptions / policy / limits / roth-401k : catchup | Additional elective deferral limit for members age 50–59 and 64+. Shared limit with traditional 401k catchup. |
| assumptions / policy / limits / roth-401k : super-catchup | Additional elective deferral limit for members age 60–63 (SECURE 2.0). Shared limit with traditional 401k super-catchup. |
| assumptions / policy / limits / roth-401k : mega | After-tax contribution limit within the annual additions limit, net of employee elective deferrals and employer contributions. |
| assumptions / policy / limits / roth-ira / phase-out | MAGI phase-out range for direct Roth IRA contributions. Adjusts annually by legislation. |
| assumptions / policy / limits / roth-ira / phase-out : floor-mfj | Lower bound of phase-out range for married filing jointly. |
| assumptions / policy / limits / roth-ira / phase-out : floor-single | Lower bound of phase-out range for single filers. |
| assumptions / policy / limits / roth-ira / phase-out : width-mfj | Width of phase-out range for married filing jointly. |
| assumptions / policy / limits / roth-ira / phase-out : width-single | Width of phase-out range for single filers. |
| assumptions / policy / limits / ira | Contribution limits for traditional and Roth IRA accounts. |
| assumptions / policy / limits / ira : max | Annual combined contribution limit across traditional and Roth IRA for a single member. |
| assumptions / policy / limits / ira : catchup | Additional contribution limit for members age 50+. Adjusted in $100 increments; not inflation-indexed. |
| assumptions / policy / limits / ira / deductibility-phase-out | MAGI phase-out range for traditional IRA deductibility. Adjusts annually by legislation. |
| assumptions / policy / limits / ira / deductibility-phase-out : floor-mfj | Lower bound of deductibility phase-out for married filing jointly. |
| assumptions / policy / limits / ira / deductibility-phase-out : floor-single | Lower bound of deductibility phase-out for single filers. |
| assumptions / policy / limits / ira / deductibility-phase-out : width-mfj | Width of deductibility phase-out range for married filing jointly. |
| assumptions / policy / limits / ira / deductibility-phase-out : width-single | Width of deductibility phase-out range for single filers. |
| assumptions / policy / limits / hsa | Contribution limits for HSA accounts. |
| assumptions / policy / limits / hsa : individual | Annual contribution limit for individual HDHP coverage. |
| assumptions / policy / limits / hsa : family | Annual contribution limit for family HDHP coverage. |
| assumptions / policy / limits / hsa : catchup | Additional contribution limit for members age 55+. Fixed amount; never inflation-adjusted. |
| assumptions / policy / rmd | RMD regulatory parameters. |
| assumptions / policy / rmd / start-age | Birth-year-to-RMD-start-age mapping table per SECURE 2.0. |
| assumptions / policy / rmd / uniform-lifetime-table | IRS Uniform Lifetime Table (Table III) — maps member age to distribution period divisor. |
| assumptions / policy / rmd / joint-survivor-table | IRS Joint Life and Last Survivor Table (Table II) — used when sole beneficiary is a spouse more than 10 years younger. |
| assumptions / policy / rmd / penalty-rate | Penalty rate for RMD shortfall. Default 25%; 10% if corrected timely per SECURE 2.0. |
| assumptions / policy / ss | Social Security regulatory parameters. |
| assumptions / policy / ss / fra | Full retirement age by birth year per SSA rules. |
| assumptions / policy / ss / delayed-credit-rate | Benefit increase rate per year of delayed claiming past FRA, up to age 70. |
| assumptions / policy / ss / taxation-thresholds | Combined income thresholds determining taxable percentage of SS benefits. Not inflation-indexed. |
| assumptions / policy / ss / taxation-thresholds : floor-mfj | Lower combined-income threshold for MFJ (below which 0% is taxable). |
| assumptions / policy / ss / taxation-thresholds : ceiling-mfj | Upper combined-income threshold for MFJ (above which up to 85% is taxable). |
| assumptions / policy / ss / taxation-thresholds : floor-single | Lower combined-income threshold for single filers. |
| assumptions / policy / ss / taxation-thresholds : ceiling-single | Upper combined-income threshold for single filers. |
| assumptions / policy / tax | Federal tax policy parameters. |
| assumptions / policy / tax / federal / brackets | Federal income tax brackets — rates and income thresholds per year, by filing status. |
| assumptions / policy / tax / federal / standard-deduction | Standard deduction amount per year by filing status. Inflation-adjusted. |
| assumptions / policy / tax / capital-gains / brackets | Long-term capital gains tax brackets — rates and income thresholds per year. |
| assumptions / policy / tax / niit / rate | Net investment income tax rate. Default 3.8%. |
| assumptions / policy / tax / niit / threshold | NIIT income thresholds by filing status. Not inflation-indexed. |
| assumptions / policy / tax / niit / threshold : mfj | NIIT income threshold for married filing jointly. |
| assumptions / policy / tax / niit / threshold : single | NIIT income threshold for single filers. |
| assumptions / policy / tax / se / rate | Self-employment tax rates — Social Security and Medicare components. |
| assumptions / policy / tax / se / wage-base | Social Security wage base for SE tax. Inflation-indexed annually. |
| assumptions / policy / tax / state / rate | State income tax rate. Defaults to 0. User-configurable. |
| assumptions / policy / medicare / eligibility-age | Age at which a household member becomes eligible for Medicare. |
| assumptions / policy / medicare / irmaa / tiers | IRMAA income tiers and per-person monthly surcharge amounts for Parts B and D. Adjustable by statute. |
| assumptions / policy / aca / fpl | Federal poverty level by household size. Updated annually by HHS. |
| assumptions / policy / aca / subsidy-cliff | Income threshold as percentage of FPL above which ACA subsidies phase out. Default 400%. |
| assumptions / policy / estate / exclusion | Federal estate tax exclusion amount. Per-decedent; supports portability multiplier for surviving spouse. |

## S: Household

| Path | Description |
|------|-------------|
| household | The people who play roles in the financial plan. A household has one or more members. |
| household / [ member ] | A stream representing an individual person in the household. Each member has its own timeline stream spanning birth year to death year. |
| household / [ member ] / relations | Relationships this member has with other members. Relationships are streams because they can change over time. |
| household / [ member ] / relations / spouse | A spousal relationship with another member. |
| household / [ member ] / relations / dependent | A dependent relationship with another member. |

## S: Assets

| Path | Description |
|------|-------------|
| assets | Things of value owned by the household. |
| assets / accounts | A financial account holds a balance that changes over time via contributions, withdrawals, growth, and conversions. Each account has a type that determines its tax treatment and regulatory constraints. |
| assets / accounts / [ member ] / [ 401k ] | A traditional 401k account. |
| assets / accounts / [ member ] / [ roth-401k ] | A Roth 401k account. |
| assets / accounts / [ member ] / [ ira ] | A traditional IRA account. |
| assets / accounts / [ member ] / [ roth-ira ] | A Roth IRA account. |
| assets / accounts / [ member ] / [ hsa ] | A health savings account. |
| assets / accounts / [ member ] / [ bank ] | A per-member bank account (checking, savings). |
| assets / accounts / [ member ] / [ brokerage ] | A per-member taxable brokerage account. |
| assets / accounts / [ bank ] | A joint or household bank account (checking, savings). |
| assets / accounts / [ brokerage ] | A joint or household taxable brokerage account. |
| assets / hard-assets | A hard asset is a non-liquid store of value that appreciates or depreciates. Not part of the liquid retirement pool. |
| assets / hard-assets / properties / [ home ] | A primary residence. |
| assets / hard-assets / properties / [ vacation-home ] | A vacation or secondary home. |
| assets / hard-assets / properties / [ parcel ] | An individual land parcel. |
| assets / hard-assets / [ vehicle ] | An individual vehicle. |
| assets / [ business ] | A business entity that generates income and/or holds equity value. |

## S: Contributions

| Path | Description |
|------|-------------|
| contributions | Money flowing into accounts. |
| contributions / [ member ] / [ 401k ] / employee | An employee contribution to a 401k. |
| contributions / [ member ] / [ 401k ] / catchup | An employee catchup contribution to a 401k. |
| contributions / [ member ] / [ 401k ] / employer | An employer contribution to a 401k. |
| contributions / [ member ] / [ roth-401k ] / employee | An employee contribution to a Roth 401k. |
| contributions / [ member ] / [ roth-401k ] / catchup | An employee catchup contribution to a Roth 401k. |
| contributions / [ member ] / [ roth-401k ] / employer | An employer contribution to a Roth 401k. |
| contributions / [ member ] / [ roth-401k ] / mega | An employee mega (post-tax) contribution to a Roth 401k. |
| contributions / [ member ] / [ hsa ] / employee | An employee contribution to an HSA. |
| contributions / [ member ] / [ hsa ] / employer | An employer contribution to an HSA. |
| contributions / [ member ] / [ ira ] / standard | A member contribution to an IRA. |
| contributions / [ member ] / [ ira ] / catchup | A member catchup contribution to an IRA. |
| contributions / [ member ] / [ roth-ira ] / standard | A member contribution to a Roth IRA. |
| contributions / [ member ] / [ roth-ira ] / catchup | A member catchup contribution to a Roth IRA. |
| contributions / [ member ] / [ bank ] | A deposit to a per-member bank account. |
| contributions / [ member ] / [ brokerage ] | A deposit to a per-member brokerage account. |
| contributions / [ bank ] | A deposit to a joint or household bank account. |
| contributions / [ brokerage ] | A deposit to a joint or household brokerage account. |

## S: Withdrawals

| Path | Description |
|------|-------------|
| withdrawals | Money flowing out of accounts. |
| withdrawals / [ member ] / [ 401k ] / standard | A withdrawal from a 401k. |
| withdrawals / [ member ] / [ 401k ] / rmd | Required minimum distribution from a 401k. |
| withdrawals / [ member ] / [ roth-401k ] | A withdrawal from a Roth 401k. |
| withdrawals / [ member ] / [ hsa ] / qualified | A qualified withdrawal from an HSA for eligible medical expenses. |
| withdrawals / [ member ] / [ hsa ] / unqualified | An unqualified withdrawal from an HSA. |
| withdrawals / [ member ] / [ ira ] / standard | A withdrawal from an IRA. |
| withdrawals / [ member ] / [ ira ] / rmd | Required minimum distribution from an IRA. |
| withdrawals / [ member ] / [ roth-ira ] | A withdrawal from a Roth IRA. |
| withdrawals / [ member ] / [ bank ] | A withdrawal from a per-member bank account. |
| withdrawals / [ member ] / [ brokerage ] | A withdrawal from a per-member brokerage account. |
| withdrawals / [ bank ] | A withdrawal from a joint or household bank account. |
| withdrawals / [ brokerage ] | A withdrawal from a joint or household brokerage account. |
| withdrawals / roth-conversion / [ member ] | A Roth conversion — withdrawal from a traditional account and contribution to a Roth account for a specific member. |

## S: Liabilities

| Path | Description |
|------|-------------|
| liabilities | Obligations owed by the household. |
| liabilities / [ amortized-loan ] | A loan with a fixed amortization schedule. |
| liabilities / [ interest-only-loan ] | An interest-only obligation (e.g. HELOC). |
| liabilities / [ credit-line ] | A revolving credit obligation (credit cards, etc). |

## S: Income

| Path | Description |
|------|-------------|
| income | An income source produces income over an active period. |
| income / w2 / [ member ] / [ employer ] | W-2 wage and salary income from a specific employer for a specific member. |
| income / 1099 / [ member ] / [ source ] | 1099 self-employment or contract income from a specific source for a specific member. |
| income / ss / [ member ] | Social Security benefit stream for a specific member. Claiming age and benefit amount vary per member. |
| income / pension / [ member ] | Defined-benefit pension income for a specific member. |
| income / k1 / [ member ] / [ entity ] | K-1 pass-through income from a specific business entity for a specific member. |
| income / 1099-r / [ member ] | 1099-R retirement account distribution income for a specific member. |
| income / rental / [ member ] / [ property ] | Rental income from a specific property for a specific member. |

## S: Expenses

| Path | Description |
|------|-------------|
| expenses | An expense consumes funds over an active period. Categorized for tax treatment and account-matching purposes. |
| expenses / housing | Housing-related expenses (property tax, maintenance, insurance, HOA). Excludes mortgage principal/interest which is a liability. |
| expenses / health | Health-related expenses. Subject to healthcare cost inflation. Eligible for HSA matching. |
| expenses / health / insurance | Health insurance premiums. Phases by life stage: employer-sponsored, ACA marketplace, and Medicare. |
| expenses / health / insurance / employer | Employer-sponsored health insurance premiums. |
| expenses / health / insurance / aca | ACA marketplace premiums (early retirement to age 65). |
| expenses / health / insurance / medicare | Medicare premiums (Parts B/D, supplemental, IRMAA surcharges). |
| expenses / health / out-of-pocket | Out-of-pocket medical costs. |
| expenses / health / long-term-care | Long-term care insurance premiums. |
| expenses / [ insurance-policy ] | A non-health insurance policy (auto, life, umbrella). |
| expenses / living | Essential living expenses (food, utilities, transportation). |
| expenses / discretionary | Discretionary spending (travel, entertainment, hobbies). |
| expenses / [ custom ] | A user-defined expense stream. |
| expenses / [ member ] / education / [ enrollment ] | Education costs for a specific enrollment period (tuition, fees, supplies). |
| expenses / [ member ] / childcare | Childcare or daycare costs for a specific member. |
| expenses / [ member ] / activities | Extracurricular activities, camps, lessons for a specific member. |
| expenses / [ member ] / personal-care | Personal care expenses for a specific member (clothing, grooming, personal supplies). |
