# Data Integrity Controls

This document catalogs all controls that prevent write, delete, or modification of records based on set conditions across all insurance modules, ensuring data integrity and consistency.

## Numbering Scheme

- **C-XX** — Control defined in the base module (`optimum_insurance_base`)
- **C-XX-01** — Extension of that control in a type-specific module (dash + sub-number)
- Controls unique to a type module get their own top-level number

---

## 1. Delete Prevention (`unlink()` overrides)

### C-01 — Cannot delete policy with endorsements

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~491) |
| **Condition** | Policy has one or more endorsements |
| **Error** | _"You cannot delete policy %s because it has %d endorsement(s) associated with it. Delete the endorsements first or cancel the policy instead."_ |

### C-02 — Cannot delete corporate client with active employees

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~229) |
| **Condition** | Corporate client has active employees (no end date or end date in the future) |
| **Error** | _"Cannot delete corporate client '%s' with %s active employee(s). Please terminate all employments first."_ |

### C-112 — Cannot delete offer with a parent offer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~906) |
| **Condition** | Offer has a `previous_offer_id` (i.e., it is a child offer or negotiation response) |
| **Error** | _"You cannot delete offer '%s' because it has a parent offer. Delete child offers first or archive the offer instead."_ |

### C-03 — Vehicle removal from offer cascades inspection cleanup

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `offer.has.vehicle` |
| **File** | `models/offer_has_vehicle.py` (line ~160) |
| **Condition** | Always on delete — removes related `vehicle.inspection.line` and orphaned inspections |
| **Type** | Cascading cleanup (no error raised) |

---

## 2. Write Validation (`write()` overrides)

### C-04 — Offer company restriction validation on write

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~884) |
| **Condition** | When `insurance_company_id` or `insurance_request_id` is changed |
| **Behavior** | Calls `_validate_company_restrictions()` before write to enforce company-level restrictions |

### C-05 — Corporate client enforces is_company on partner

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~215) |
| **Condition** | After any write |
| **Behavior** | Ensures `partner_id.is_company` remains `True` |

### C-06 — Shipment request trip type field validation on write

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `insurance.request.shipment` |
| **File** | `models/insurance_request_shipment.py` (line ~148) |
| **Condition** | On create and write — validates required location fields based on trip type |
| **Errors** | _"Export From Location is required for Export Only and Export & Import trip types!"_, _"Export From Country is required when Export From Location is Specific Country!"_, and similar for all import/export location combinations |

---

## 3. Python Constraints (`@api.constrains`)

### C-07 — Policy commission validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~452) |
| **Constrains** | `is_commission_exception`, `commission_rate`, `contract_id` |
| **Errors** | _"For manual commission policies, you must specify a commission rate greater than 0."_ / _"No active contract found for company '%s' on date %s."_ |

### C-08 — Policy payment schedule must equal gross premium

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` (line ~476) |
| **Constrains** | `payment_schedule_ids`, `gross_premium` |
| **Error** | _"Payment schedule total (%(total)s) must equal the gross premium (%(premium)s)."_ |

### C-09 — Offer negotiation scenario must have parent offer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.offer` |
| **File** | `models/insurance_offer.py` (line ~677) |
| **Constrains** | `is_negotiation_scenario`, `previous_offer_id` |
| **Errors** | _"A negotiation scenario must have a parent offer."_ / _"A negotiation scenario's parent must be an actual offer, not another scenario."_ |

#### C-09-01 — Medical offer skips base coverage check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `insurance.offer` (inherited) |
| **File** | `models/insurance_offer.py` (line ~104) |
| **Constrains** | `coverage_ids` |
| **Behavior** | Overrides base `_check_has_coverage` — skips coverage check for medical offers (uses medical categories instead) |

### C-10 — Client must have exactly one specialized record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client` |
| **File** | `models/client.py` (line ~123) |
| **Constrains** | `individual_id`, `corporate_id` |
| **Errors** | _"Client '%s' must have exactly one specialized record (individual or corporate). Currently has none."_ / _"Client '%s' must have exactly one specialized record. Current: %s individual, %s corporate"_ / Type mismatch errors |

### C-11 — Client area must belong to district, district must belong to state

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client` |
| **File** | `models/client.py` (line ~172) |
| **Constrains** | `area_id`, `district_id`, `state_id` |
| **Errors** | _"Area '%s' does not belong to district '%s'."_ / _"District '%s' does not belong to state '%s'."_ |

#### C-11-01 — CRM lead geographic validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `crm.lead` |
| **File** | `models/crm_lead.py` (line ~231) |
| **Constrains** | `district_id`, `state_id`, `area_id` |
| **Errors** | _"District '%s' does not belong to state '%s'."_ / _"Area '%s' does not belong to district '%s'."_ |

### C-12 — Corporate client disjoint type check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` (line ~118) |
| **Constrains** | `client_id` |
| **Error** | _"Client '%s' is already registered as an individual client. A client cannot be both individual and corporate."_ |

### C-13 — Individual client disjoint type check

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` (line ~255) |
| **Constrains** | `client_id` |
| **Error** | _"Client '%s' is already registered as a corporate client. A client cannot be both individual and corporate."_ |

### C-14 — Individual birth date cannot be in the future

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` (line ~266) |
| **Constrains** | `birth_date` |
| **Error** | _"Birth date cannot be in the future. Please enter a valid date of birth."_ |

### C-15 — CRM lead email validation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `crm.lead` |
| **File** | `models/crm_lead.py` (line ~200) |
| **Constrains** | `email_from` |
| **Errors** | _"Email cannot be a website URL (starting with 'www')."_ / _"Invalid email format."_ / _"Email domain '%s' is not allowed."_ |

### C-16 — Insurance company scores must be between 0 and 5

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` (line ~289) |
| **Constrains** | All 12 score fields (medical/general/motor prices, claims, endorsements, SLA) |
| **Error** | _"%s must be between 0 and 5. Current value: %s"_ |

### C-17 — Employment end date cannot be before start date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~192) |
| **Constrains** | `start_date`, `end_date` |
| **Error** | _"Employment end date (%s) cannot be before start date (%s)"_ |

### C-18 — No overlapping employment periods

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~203) |
| **Constrains** | `corporate_client_id`, `individual_id`, `start_date`, `end_date` |
| **Error** | _"Employment period overlaps with existing employment..."_ |

### C-19 — Insurance request must have client or CRM lead

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~336) |
| **Constrains** | `client_id`, `crm_lead_id` |
| **Error** | _"Either a Client or a CRM Lead must be specified for the insurance request."_ |

### C-20 — Renewal request must have trigger

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~346) |
| **Constrains** | `is_renewal`, `renewal_trigger` |
| **Error** | _"Renewal trigger is required for renewal requests."_ |

### C-21 — Renewal request cannot have CRM lead

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~355) |
| **Constrains** | `is_renewal`, `crm_lead_id` |
| **Error** | _"CRM Lead must be empty for renewal requests."_ |

### C-22 — Offer deadline must be after request date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.request` |
| **File** | `models/insurance_request.py` (line ~364) |
| **Constrains** | `offer_submission_deadline`, `request_date` |
| **Error** | _"The offer submission deadline must be after the insurance request date."_ |

### C-23 — Copay percentage must be between 0 and 100

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.category.has.benefit.type` |
| **File** | `models/offer_category_has_benefit_type.py` (line ~66) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-01 — Copay percentage on offer coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~90) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-02 — Copay percentage on policy benefit types

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.category.has.benefit.type` |
| **File** | `models/policy_category_has_benefit_type.py` (line ~88) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

#### C-23-03 — Copay percentage on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~90) |
| **Constrains** | `copay_percentage` |
| **Error** | _"Copay percentage must be between 0 and 100."_ |

### C-24 — Coverage item parent must match benefit type line

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~65) |
| **Constrains** | `coverage_id`, `benefit_type_line_id` |
| **Error** | _"Coverage item '%(item)s' belongs to '%(item_parent)s' but is assigned under '%(line_parent)s'."_ |

#### C-24-01 — Same check on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~65) |
| **Constrains** | `coverage_id`, `benefit_type_line_id` |
| **Error** | Same as C-24 |

### C-25 — Coverage item limit type consistency

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.benefit.has.coverage.item` |
| **File** | `models/offer_benefit_has_coverage_item.py` (line ~78) |
| **Constrains** | `limit_type`, `limit_amount`, `limit_quantity` |
| **Error** | _"Limit type '%s' requires a limit amount/quantity."_ |

#### C-25-01 — Same check on policy coverage items

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.benefit.has.coverage.item` |
| **File** | `models/policy_benefit_has_coverage_item.py` (line ~78) |
| **Constrains** | `limit_type`, `limit_amount`, `limit_quantity` |
| **Error** | Same as C-25 |

### C-26 — Insurable item must belong to correct client (client match)

This pattern is replicated across all type-specific junction tables:

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `offer.has.employee` |
| **File** | `models/offer_has_employee.py` (line ~36) |
| **Constrains** | `employee_id`, `offer_id` |
| **Error** | _"Employee '%s' does not belong to the offer's client."_ |

#### C-26-01 — Employee-policy client match (medical)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `policy.has.employee` |
| **File** | `models/policy_has_employee.py` (line ~38) |
| **Error** | _"Employee '%s' does not belong to the policy holder."_ |

#### C-26-02 — Employee-request client match (medical)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **Model** | `request.has.employee` |
| **File** | `models/request_has_employee.py` (line ~35) |
| **Error** | _"Employee '%s' does not belong to the request's client."_ |

#### C-26-03 — Vehicle-offer client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `offer.has.vehicle` |
| **File** | `models/offer_has_vehicle.py` (line ~177) |
| **Error** | _"Vehicle '%s' does not belong to the offer's client."_ |

#### C-26-04 — Vehicle-policy client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `policy.has.vehicle` |
| **File** | `models/policy_has_vehicle.py` (line ~55) |
| **Error** | _"Vehicle '%s' does not belong to the policy holder."_ |

#### C-26-05 — Vehicle-request client match (vehicle)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `request.has.vehicle` |
| **File** | `models/request_has_vehicle.py` (line ~52) |
| **Error** | _"Vehicle '%s' does not belong to the request's client."_ |

#### C-26-06 — Shipment-offer client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `offer.has.shipment` |
| **File** | `models/offer_has_shipment.py` (line ~57) |
| **Error** | _"Shipment '%s' does not belong to the offer's client."_ |

#### C-26-07 — Shipment-policy client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `policy.has.shipment` |
| **File** | `models/policy_has_shipment.py` (line ~45) |
| **Error** | _"Shipment '%s' does not belong to the policy holder."_ |

#### C-26-08 — Shipment-request client match (shipment)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `request.has.shipment` |
| **File** | `models/request_has_shipment.py` (line ~42) |
| **Error** | _"Shipment '%s' does not belong to the request's client."_ |

#### C-26-09 — Estate-offer client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `offer.has.estate` |
| **File** | `models/offer_has_estate.py` (line ~58) |
| **Error** | _"Estate '%s' does not belong to the offer's client."_ |

#### C-26-10 — Estate-policy client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `policy.has.estate` |
| **File** | `models/policy_has_estate.py` (line ~46) |
| **Error** | _"Estate '%s' does not belong to the policy holder."_ |

#### C-26-11 — Estate-request client match (general)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `request.has.estate` |
| **File** | `models/request_has_estate.py` (line ~43) |
| **Error** | _"Estate '%s' does not belong to the request's client."_ |

---

## 4. SQL Constraints (`models.Constraint`)

### 4a. UNIQUE Constraints

### C-27 — Policy number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy` |
| **File** | `models/insurance_policy.py` |
| **SQL** | `UNIQUE(policy_number)` |
| **Error** | _"Policy number must be unique!"_ |

### C-28 — Endorsement number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` |
| **SQL** | `UNIQUE(endorsement_number)` |
| **Error** | _"Endorsement number must be unique!"_ |

### C-29 — Insurance company partner must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` |
| **SQL** | `UNIQUE(partner_id)` |
| **Error** | _"A partner can only be associated with one insurance company."_ |

### C-30 — Insurance company name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.company` |
| **File** | `models/insurance_company.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Insurance company name must be unique!"_ |

### C-31 — Each client can have only one corporate record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.corporate` |
| **File** | `models/client_corporate.py` |
| **SQL** | `UNIQUE(client_id)` |
| **Error** | _"Each client can have only one corporate record!"_ |

### C-32 — Each client can have only one individual record

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` |
| **SQL** | `UNIQUE(client_id)` |
| **Error** | _"Each client can have only one individual record!"_ |

### C-33 — Individual ID number must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `client.individual` |
| **File** | `models/client_individual.py` |
| **SQL** | `UNIQUE(id_number)` |
| **Error** | _"ID number must be unique!"_ |

### C-34 — Employment start date unique per individual and corporate

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` |
| **SQL** | `UNIQUE(corporate_client_id, individual_id, start_date)` |
| **Error** | _"Each individual can only have one employment with the same company starting on the same date!"_ |

### C-35 — Claim denial reason name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `claim.denial.reason` |
| **File** | `models/claim_denial_reason.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Claim denial reason must be unique!"_ |

### C-36 — Unique insurable item per offer/policy/request

This pattern is replicated across all type-specific junction tables:

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-36** | `medical` | `offer.has.employee` | `UNIQUE(offer_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per offer and category!"_ |
| **C-36-01** | `medical` | `policy.has.employee` | `UNIQUE(policy_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per policy and category!"_ |
| **C-36-02** | `medical` | `request.has.employee` | `UNIQUE(insurance_request_id, employee_id, COALESCE(category_id, 0))` | _"Each employee can only be added once per request and category!"_ |
| **C-36-03** | `vehicle` | `offer.has.vehicle` | `UNIQUE(offer_id, vehicle_id)` | _"Each vehicle can only be added once per offer!"_ |
| **C-36-04** | `vehicle` | `policy.has.vehicle` | `UNIQUE(policy_id, vehicle_id)` | _"Each vehicle can only be added once per policy!"_ |
| **C-36-05** | `vehicle` | `request.has.vehicle` | `UNIQUE(insurance_request_id, vehicle_id)` | _"Each vehicle can only be added once per request!"_ |
| **C-36-06** | `shipment` | `offer.has.shipment` | `UNIQUE(offer_id, shipment_id)` | _"Each shipment can only be added once per offer!"_ |
| **C-36-07** | `shipment` | `policy.has.shipment` | `UNIQUE(policy_id, shipment_id)` | _"Each shipment can only be added once per policy!"_ |
| **C-36-08** | `shipment` | `request.has.shipment` | `UNIQUE(insurance_request_id, shipment_id)` | _"Each shipment can only be added once per request!"_ |
| **C-36-09** | `general` | `offer.has.estate` | `UNIQUE(offer_id, estate_id)` | _"Each estate can only be added once per offer!"_ |
| **C-36-10** | `general` | `policy.has.estate` | `UNIQUE(policy_id, estate_id)` | _"Each estate can only be added once per policy!"_ |
| **C-36-11** | `general` | `request.has.estate` | `UNIQUE(insurance_request_id, estate_id)` | _"Each estate can only be added once per request!"_ |

### C-37 — Medical category number unique per parent

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-37** | `medical` | `insurance.offer.medical.category` | `UNIQUE(offer_id, category_number)` | _"Category number must be unique per offer!"_ |
| **C-37-01** | `medical` | `insurance.request.medical.category` | `UNIQUE(insurance_request_id, category_number)` | _"Category number must be unique per request!"_ |
| **C-37-02** | `medical` | `insurance.policy.medical.category` | `UNIQUE(policy_id, category_number)` | _"Category number must be unique per policy!"_ |

### C-38 — Unique benefit type per medical category

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-38** | `medical` | `offer.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |
| **C-38-01** | `medical` | `policy.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |
| **C-38-02** | `medical` | `request.category.has.benefit.type` | `UNIQUE(category_id, coverage_id)` | _"Each benefit type can only appear once per category!"_ |

### C-39 — Unique coverage item per benefit type line

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-39** | `medical` | `offer.benefit.has.coverage.item` | `UNIQUE(benefit_type_line_id, coverage_id)` | _"Each coverage item can only appear once per benefit type line!"_ |
| **C-39-01** | `medical` | `policy.benefit.has.coverage.item` | `UNIQUE(benefit_type_line_id, coverage_id)` | _"Each coverage item can only appear once per benefit type line!"_ |

### C-40 — Vehicle inspection line unique per inspection

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.inspection.line` |
| **File** | `models/vehicle_inspection_line.py` |
| **SQL** | `UNIQUE(inspection_id, vehicle_id)` |
| **Error** | _"Each vehicle can only appear once per inspection!"_ |

### C-41 — Vehicle model name unique per manufacturer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.model` |
| **File** | `models/vehicle_model.py` |
| **SQL** | `UNIQUE(name, manufacturer_id)` |
| **Error** | _"Model name must be unique per manufacturer!"_ |

### C-42 — Vehicle manufacturer name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.manufacturer` |
| **File** | `models/vehicle_manufacturer.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Manufacturer name must be unique!"_ |

### C-43 — Vehicle part name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.part` |
| **File** | `models/vehicle_part.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Vehicle part name must be unique!"_ |

### C-44 — Year of production unique per vehicle model

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `UNIQUE(vehicle_model_id, year)` |
| **Error** | _"Year of production must be unique per model!"_ |

### C-45 — Service centre unique per manufacturer

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.manufacturer.service.centre` |
| **File** | `models/vehicle_manufacturer_service_centre.py` |
| **SQL** | `UNIQUE(service_centre_id, vehicle_manufacturer_id)` |
| **Error** | _"Service centre is already linked to this manufacturer!"_ |

### C-46 — One shipment record per insurance request

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `insurance.request.shipment` |
| **File** | `models/insurance_request_shipment.py` |
| **SQL** | `UNIQUE(request_id)` |
| **Error** | _"Only one shipment record is allowed per insurance request!"_ |

### C-47 — Packing method name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **Model** | `shipment.packing.method` |
| **File** | `models/shipment_packing_method.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Packing method name must be unique!"_ |

### C-48 — Estate type name must be unique

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **Model** | `estate.type` |
| **File** | `models/estate_type.py` |
| **SQL** | `UNIQUE(name)` |
| **Error** | _"Estate type name must be unique!"_ |

### 4b. CHECK Constraints

### C-49 — Contract early payment reward days must be positive

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `contract.early.payment.reward` |
| **File** | `models/contract_early_payment_reward.py` |
| **SQL** | `CHECK(days_valid > 0)` |
| **Error** | _"Days valid must be greater than zero!"_ |

### C-50 — Employment end date >= start date (SQL level)

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` |
| **SQL** | `CHECK(end_date IS NULL OR end_date >= start_date)` |
| **Error** | _"Employment end date must be on or after start date!"_ |

### C-51 — Medical category number must be positive

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-51** | `medical` | `insurance.offer.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |
| **C-51-01** | `medical` | `insurance.request.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |
| **C-51-02** | `medical` | `insurance.policy.medical.category` | `CHECK(category_number > 0)` | _"Category number must be positive!"_ |

### C-52 — Benefit type annual limit cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-52** | `medical` | `offer.category.has.benefit.type` | `CHECK(annual_limit >= 0 OR annual_limit IS NULL)` | _"Benefit type annual limit cannot be negative!"_ |
| **C-52-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(annual_limit >= 0 OR annual_limit IS NULL)` | _"Benefit type annual limit cannot be negative!"_ |

### C-53 — Waiting period months cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-53** | `medical` | `offer.category.has.benefit.type` | `CHECK(waiting_period_months >= 0 OR waiting_period_months IS NULL)` | _"Waiting period months cannot be negative!"_ |
| **C-53-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(waiting_period_months >= 0 OR waiting_period_months IS NULL)` | _"Waiting period months cannot be negative!"_ |

### C-54 — Additional premium cannot be negative

| Sub-Control | Module | Model | SQL | Error |
|-------------|--------|-------|-----|-------|
| **C-54** | `medical` | `offer.category.has.benefit.type` | `CHECK(additional_premium >= 0 OR additional_premium IS NULL)` | _"Additional premium cannot be negative!"_ |
| **C-54-01** | `medical` | `policy.category.has.benefit.type` | `CHECK(additional_premium >= 0 OR additional_premium IS NULL)` | _"Additional premium cannot be negative!"_ |

### C-55 — Vehicle price must be positive

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(price >= 0)` |
| **Error** | _"Price must be positive!"_ |

### C-56 — Vehicle minimum price must not exceed maximum

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(minimum_price <= maximum_price)` |
| **Error** | _"Minimum price must not exceed maximum price!"_ |

### C-57 — Vehicle average price must be within range

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **Model** | `vehicle.year.of.prod` |
| **File** | `models/vehicle_year_of_prod.py` |
| **SQL** | `CHECK(minimum_price <= price AND price <= maximum_price)` |
| **Error** | _"Average price must be between minimum and maximum price!"_ |

---

## 5. Foreign Key Restrictions (`ondelete='restrict'`)

These prevent deletion of parent records when child records reference them.

### Base Module (`optimum_insurance_base`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-58** | `insurance.policy` | `policy_holder_id` | `client` with policies | `models/insurance_policy.py` |
| **C-59** | `insurance.policy` | `offer_id` | `insurance.offer` with policies | `models/insurance_policy.py` |
| **C-60** | `insurance.policy` | `insurance_type_id` | `insurance.type` with policies | `models/insurance_policy.py` |
| **C-61** | `insurance.offer` | `insurance_request_id` | `insurance.request` with offers | `models/insurance_offer.py` |
| **C-62** | `insurance.offer` | `insurance_type_id` | `insurance.type` with offers | `models/insurance_offer.py` |
| **C-63** | `insurance.policy.endorsement` | `policy_id` | `insurance.policy` with endorsements | `models/insurance_policy_endorsement.py` |
| **C-64** | `crm.lead` | `district_id` | `res.country.state.district` with leads | `models/crm_lead.py` |
| **C-65** | `crm.lead` | `area_id` | `res.country.state.district.area` with leads | `models/crm_lead.py` |
| **C-66** | `client` | `district_id` | `res.country.state.district` with clients | `models/client.py` |
| **C-67** | `client` | `area_id` | `res.country.state.district.area` with clients | `models/client.py` |
| **C-68** | `client.corporate` | `industry_id` | `res.partner.industry` with corporates | `models/client_corporate.py` |
| **C-69** | `corporate.client.individual` | `insurance_type_id` | `insurance.type` with employments | `models/corporate_client_individual.py` |
| **C-70** | `insurance.company.has.employee` | `insurance_type_id` | `insurance.type` with company employees | `models/insurance_company_employee.py` |

### Medical Module (`optimum_insurance_medical`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-71** | `offer.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with benefit types | `models/offer_category_has_benefit_type.py` |
| **C-72** | `offer.benefit.has.coverage.item` | `coverage_id` | `insurance.coverage` with coverage items | `models/offer_benefit_has_coverage_item.py` |
| **C-73** | `offer.has.employee` | `employee_id` | `client.insurable.employee` in offers | `models/offer_has_employee.py` |
| **C-74** | `policy.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with policy benefits | `models/policy_category_has_benefit_type.py` |
| **C-75** | `policy.benefit.has.coverage.item` | `coverage_id` | `insurance.coverage` with policy items | `models/policy_benefit_has_coverage_item.py` |
| **C-76** | `policy.has.employee` | `employee_id` | `client.insurable.employee` in policies | `models/policy_has_employee.py` |
| **C-77** | `request.category.has.benefit.type` | `coverage_id` | `insurance.coverage` with request benefits | `models/request_category_has_benefit_type.py` |
| **C-78** | `request.has.employee` | `employee_id` | `client.insurable.employee` in requests | `models/request_has_employee.py` |
| **C-79** | `client.insurable.employee` | `client_id` | `client` with insurable employees | `models/client_insurable_employee.py` |
| **C-80** | `offer.beneficiary` | `employee_id` | `client.insurable.employee` as beneficiaries | `models/offer_beneficiary.py` |
| **C-81** | `policy.beneficiary` | `employee_id` | `client.insurable.employee` as beneficiaries | `models/policy_beneficiary.py` |

### Vehicle Module (`optimum_insurance_vehicle`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-82** | `offer.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in offers | `models/offer_has_vehicle.py` |
| **C-83** | `policy.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in policies | `models/policy_has_vehicle.py` |
| **C-84** | `request.has.vehicle` | `vehicle_id` | `client.insurable.vehicle` in requests | `models/request_has_vehicle.py` |
| **C-85** | `vehicle.inspection.line` | `vehicle_id` | `client.insurable.vehicle` in inspections | `models/vehicle_inspection_line.py` |
| **C-86** | `vehicle.model` | `manufacturer_id` | `vehicle.manufacturer` with models | `models/vehicle_model.py` |
| **C-87** | `client.insurable.vehicle` | `client_id` | `client` with insurable vehicles | `models/client_insurable_vehicle.py` |
| **C-88** | `client.insurable.vehicle` | `vehicle_year_of_prod_id` | `vehicle.year.of.prod` with vehicles | `models/client_insurable_vehicle.py` |
| **C-89** | `insurance.company.service.centres` | `insurance_company_id` | `insurance.company` with service centres | `models/insurance_company_service_centres.py` |
| **C-90** | `insurance.company.service.centres` | `service_centre_id` | service centre with company authorizations | `models/insurance_company_service_centres.py` |

### Shipment Module (`optimum_insurance_shipment`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-91** | `offer.has.shipment` | `shipment_id` | `client.insurable.shipment` in offers | `models/offer_has_shipment.py` |
| **C-92** | `policy.has.shipment` | `shipment_id` | `client.insurable.shipment` in policies | `models/policy_has_shipment.py` |
| **C-93** | `request.has.shipment` | `shipment_id` | `client.insurable.shipment` in requests | `models/request_has_shipment.py` |
| **C-94** | `insurance.request.shipment` | `export_from_country_id` | `res.country` in export config | `models/insurance_request_shipment.py` |
| **C-95** | `insurance.request.shipment` | `export_from_state_id` | `res.country.state` in export config | `models/insurance_request_shipment.py` |
| **C-96** | `insurance.request.shipment` | `export_to_country_id` | `res.country` in export config | `models/insurance_request_shipment.py` |
| **C-97** | `insurance.request.shipment` | `export_to_state_id` | `res.country.state` in export config | `models/insurance_request_shipment.py` |
| **C-98** | `insurance.request.shipment` | `import_from_country_id` | `res.country` in import config | `models/insurance_request_shipment.py` |
| **C-99** | `insurance.request.shipment` | `import_from_state_id` | `res.country.state` in import config | `models/insurance_request_shipment.py` |
| **C-100** | `insurance.request.shipment` | `import_to_country_id` | `res.country` in import config | `models/insurance_request_shipment.py` |
| **C-101** | `insurance.request.shipment` | `import_to_state_id` | `res.country.state` in import config | `models/insurance_request_shipment.py` |
| **C-102** | `insurance.request.shipment` | `packing_method_id` | `shipment.packing.method` in requests | `models/insurance_request_shipment.py` |

### General Module (`optimum_insurance_general`)

| Control | Model | Field | Prevents Deletion Of | File |
|---------|-------|-------|---------------------|------|
| **C-103** | `offer.has.estate` | `estate_id` | `client.insurable.estate` in offers | `models/offer_has_estate.py` |
| **C-104** | `policy.has.estate` | `estate_id` | `client.insurable.estate` in policies | `models/policy_has_estate.py` |
| **C-105** | `request.has.estate` | `estate_id` | `client.insurable.estate` in requests | `models/request_has_estate.py` |

---

## 6. Action Method Guards

### C-106 — Cannot terminate already-terminated employment

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~269) |
| **Method** | `action_terminate_employment()` |
| **Condition** | Employment already has end date in the past |
| **Error** | _"Employment is already terminated (ended on %s)"_ |

### C-107 — Cannot extend employment with no end date

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `corporate.client.individual` |
| **File** | `models/corporate_client_individual.py` (line ~297) |
| **Method** | `action_extend_employment()` |
| **Condition** | Employment has no end date (already active indefinitely) |
| **Error** | _"Employment is already active (no end date set)"_ |

### C-108 — Cannot apply already-applied endorsement

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (line ~200) |
| **Method** | `action_apply_changes()` |
| **Condition** | Endorsement `is_applied` is already True |
| **Error** | _"This endorsement has already been applied."_ |

### C-109 — Cannot reset applied endorsement to draft

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **Model** | `insurance.policy.endorsement` |
| **File** | `models/insurance_policy_endorsement.py` (line ~252) |
| **Method** | `action_reset_to_draft()` |
| **Condition** | Endorsement `is_applied` is True |
| **Error** | _"Cannot reset an applied endorsement."_ |

---

## 7. View Readonly Controls

### 7a. Conditional Readonly (state-dependent)

These fields become readonly based on business state, preventing modification at certain stages.

### C-110 — Offer financial fields readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_base` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `insurance_request_id`, `insurance_company_id`, `insurance_type_id`, `net_premium`, `gross_premium`, `sum_insurance`, `gross_rate`, `insurance_duration`, `number_of_checks`, `coverage_ids` |
| **Purpose** | Prevents modification of core offer data after negotiation scenarios have been created |

#### C-110-01 — Medical categories readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_medical` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `medical_category_ids`, `employee_ids` |

#### C-110-02 — Vehicle list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `vehicle_ids` |

#### C-110-03 — Shipment list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_shipment` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `shipment_ids` |

#### C-110-04 — Estate list readonly when has child offers

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_general` |
| **View** | `views/insurance_offer_views.xml` |
| **Condition** | `readonly="has_child_offers"` |
| **Fields affected** | `estate_ids` |

### C-111 — Vehicle inspection body parts readonly during negotiation

| Attribute | Value |
|-----------|-------|
| **Module** | `optimum_insurance_vehicle` |
| **View** | `views/vehicle_inspection_views.xml` |
| **Condition** | `readonly="parent.inspection_state == 'negotiation'"` |
| **Fields affected** | `vehicle_part_id`, `custom_part_name`, `status`, `photo`, `damage_description` |
| **Purpose** | Prevents modification of damage assessments once inspection enters negotiation |

### 7b. Always-Readonly Fields (computed / reference / system-generated)

These fields are always `readonly="1"` because they are computed, system-generated, or reference fields that should not be manually edited.

#### Offer Form (`optimum_insurance_base` — `insurance_offer_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name` | Auto-generated offer name |
| `prev_net_premium`, `prev_gross_premium`, `prev_sum_insurance`, `prev_gross_rate`, `prev_insurance_duration`, `prev_number_of_checks` | Previous version comparison values |
| `wording_review_status`, `coverage_reviewer_id`, `wording_reviewer_id` | Review status badges |
| `inspection_id`, `previous_offer_id`, `child_offer_count` | Reference / computed |
| `mandatory_coverage_status` | Computed widget |
| `prev_coverage_*`, `prev_deductible_*`, `prev_copayment_*` fields | Previous version comparison values in coverage/deductible/copayment lines |
| `limit_review_status`, `coverage_review_notes` | Review status |
| `deductible_review_status`, `deductible_review_notes` | Review status |
| `copayment_review_status`, `copayment_review_notes` | Review status |
| Predefined descriptions, company wording, standard wording | Reference text |
| `has_copay` | Computed flag |
| `pricing_contact_ids` | Reference |

#### Policy Form (`optimum_insurance_base` — `insurance_policy_views.xml`)

| Fields | Purpose |
|--------|---------|
| `first_check_completed`, `early_payment_bonus` | Computed |
| `predefined_description`, `is_mandatory` | Reference |
| `sequence` (check #) | Auto-generated |
| `contract_id` | System-determined |

#### Endorsement Form (`optimum_insurance_base` — `insurance_policy_endorsement_views.xml`)

| Fields | Purpose |
|--------|---------|
| `endorsement_number` | Auto-generated |
| `policy_holder_id`, `insurance_company_id`, `insurance_type_id` | Inherited from policy |
| `description` | System-generated |
| `is_applied` | State toggle |

#### Request Form (`optimum_insurance_base` — `insurance_request_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name` | Auto-generated |
| Status fields, `is_confirmed`, `offer_count` | Computed |
| `declared_company_count`, `offers_received_count`, `offers_completion_percentage` | Computed statistics |

#### Inspection Form (`optimum_insurance_base` — `insurance_inspection_views.xml`)

| Fields | Purpose |
|--------|---------|
| `name`, `inspection_datetime` | Auto-generated |

#### KYC History Form (`optimum_insurance_base` — `insurance_kyc_history_views.xml`)

| Fields | Purpose |
|--------|---------|
| `policy_duration_months`, `loss_ratio`, `overall_rating` | Computed |
| `create_uid`, `create_date`, `write_date` | Audit fields |

#### Other Base Views

| View | Fields | Purpose |
|------|--------|---------|
| `client_corporate_views.xml` | `active` (in employment table) | Status display |
| `client_individual_views.xml` | `current_employer_id` | Computed |
| `corporate_client_individual_views.xml` | `display_name`, `email`, `phone`, `end_date` | Reference / computed |
| `insurance_collection_delivery_views.xml` | `name` | Auto-generated |
| `crm_lead_views.xml` | `client_contact_ids` | Reference |
| `insurance_public_terms_views.xml` | `display_name` | Computed |
| `offer_has_copayment_views.xml` | `copayment_review_status`, `copayment_review_notes` | Review status |
| `offer_has_service_views.xml` | `has_copay` | Computed |
| `policy_has_service_views.xml` | `has_copay` | Computed |
| `request_client_company_preference_views.xml` | `name` | Reference |

#### Medical Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `offer_id`, `client_id` | Derived from employee |
| `policy_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `policy_id`, `policy_holder_id`, `policy_state` | Derived from employee |
| `request_has_employee_views.xml` | `full_name`, `gender`, `relation`, `id_card_number`, `insurance_request_id`, `client_id` | Derived from employee |
| `client_insurable_employee_views.xml` | `name`, `is_insured` | Computed |
| `insurance_offer_medical_category_views.xml` | `offer_id` | Parent reference |
| `insurance_request_medical_category_views.xml` | `insurance_request_id` | Parent reference |
| `insurance_policy_medical_category_views.xml` | `policy_id` | Parent reference |

#### Vehicle Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_vehicle_views.xml` | `manufacturer_id`, `vehicle_model_id`, `number_plate`, `chassis_number`, vehicle prices, `vehicle_inspection_line_id`, `offer_id`, `client_id` | Derived from vehicle |
| `policy_has_vehicle_views.xml` | Same pattern + `policy_id`, `policy_holder_id`, `policy_state` | Derived from vehicle |
| `request_has_vehicle_views.xml` | Same pattern + `insurance_request_id`, `client_id` | Derived from vehicle |
| `client_insurable_vehicle_views.xml` | `name`, `is_insured`, `manufacturer_id`, `vehicle_model_id` | Computed |
| `vehicle_inspection_views.xml` | `inspection_id`, `number_plate`, `chassis_number`, `manufacturer_id`, `vehicle_model_id` | Reference fields in inspection lines |
| `vehicle_model_views.xml` | `normalised_name` | Computed |
| `vehicle_year_of_prod_views.xml` | `name` | Computed |

#### Shipment Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_shipment_views.xml` | `shipment_type`, `weight`, `from_country_id`, `to_country_id`, `method_of_packing_id`, `offer_id`, `client_id` | Derived from shipment |
| `policy_has_shipment_views.xml` | Same + `policy_id`, `policy_holder_id`, `policy_state` | Derived from shipment |
| `request_has_shipment_views.xml` | Same + `insurance_request_id`, `client_id` | Derived from shipment |
| `client_insurable_shipment_views.xml` | `name`, `is_insured` | Computed |

#### General Module Junction Table Views

| View | Fields | Purpose |
|------|--------|---------|
| `offer_has_estate_views.xml` | `estate_type_id`, `address`, `area_sqm`, `estimated_value`, `country_id`, `city`, `offer_id`, `client_id` | Derived from estate |
| `policy_has_estate_views.xml` | Same + `policy_id`, `policy_holder_id`, `policy_state` | Derived from estate |
| `request_has_estate_views.xml` | Same + `insurance_request_id`, `client_id` | Derived from estate |
| `client_insurable_estate_views.xml` | `name`, `is_insured` | Computed |

---

## Summary

| Category | Count |
|----------|-------|
| Delete Prevention (unlink) | 4 |
| Write Validation (write) | 3 |
| Python Constraints (@api.constrains) | 26 (with sub-controls) |
| SQL UNIQUE Constraints | 22 (with sub-controls) |
| SQL CHECK Constraints | 9 (with sub-controls) |
| Foreign Key Restrictions (ondelete restrict) | 48 |
| Action Method Guards | 4 |
| Conditional View Readonly | 2 patterns (with 4 extensions) |
| Always-Readonly View Fields | 100+ fields across all modules |
| **Total named controls** | **C-01 through C-112** |
