# Mambu Data Model – Relations

Based on the four services: `GetClient`, `GetLoan`, `GetProduct`, `GetSchedule`.

---

## Entity overview

| Entity   | Mambu resource      | Key (example)                        | API path                            |
|----------|---------------------|--------------------------------------|-------------------------------------|
| Client   | `/api/clients`      | `8a19b6a69d255395019d262ee2a572f4`   | `/api/clients/{encodedKey}`         |
| Loan     | `/api/loans`        | `8a19dc979d254c1d019d2630046f7e3d`   | `/api/loans/{encodedKey}`           |
| Product  | `/api/loanproducts` | `8a19b2ee97346d40019734c0b48d2866`   | `/api/loanproducts/{encodedKey}`    |
| Schedule | (sub-resource)      | (inherited from Loan)                | `/api/loans/{loanKey}/schedule`     |

---

## Relation diagram

```
LoanProduct (template/configuration)
    │
    │  productTypeKey
    │  (1 product → many loans)
    ▼
  Loan ◄──────────────────── Client
    │   accountHolderKey         │
    │   (1 client → many loans)  │
    │                            │
    │  assignedBranchKey ────────┘  (shared branch)
    │
    │  (1 loan → 1 payment schedule)
    ▼
 Schedule
```

---

## Relations in detail

### Client → Loan
- **Link:** `Loan.accountHolderKey = Client.encodedKey`
- **Cardinality:** A client can have **many loans**
- **Type:** `Loan.accountHolderType = "CLIENT"` (can also be GROUP)
- **Example:** Client `8a19b6a69d255395019d262ee2a572f4` owns Loan `8a19dc979d254c1d019d2630046f7e3d`

### LoanProduct → Loan
- **Link:** `Loan.productTypeKey = LoanProduct.encodedKey`
- **Cardinality:** A product is a template for **many loans**
- **Purpose:** The product defines interest calculation, repayment method, fees, and validations. The loan inherits these settings but can override them within the product's limits.
- **Example:** Product `8a19b2ee97346d40019734c0b48d2866` (`SPK Boliglån Annuitet`) → Loan `8a19dc979d254c1d019d2630046f7e3d`

### Loan → Schedule
- **Link:** Schedule is fetched via `/api/loans/{loanKey}/schedule` – there is **no separate encodedKey**; Schedule is a calculated sub-resource of Loan
- **Cardinality:** A loan has **one payment schedule**
- **Content:** 24 installment payments (annuity), due on the 15th of each month (`fixedDaysOfMonth: [15]`)
- **Note:** Schedule is **not a stored entity** – it is calculated dynamically by Mambu based on the loan's `scheduleSettings` and `interestSettings`

### Branch (shared reference)
- **Link:** Both `Client.assignedBranchKey` and `Loan.assignedBranchKey` point to `8a19c1ab966cca9901966cf9dceb29b8`
- **Purpose:** Ensures that the loan and client belong to the same branch/office in the organisation

---

## Custom Fields on Loan

The loan has three domain-specific field groups that are not part of the standard Mambu model:

| Field group                     | Fields                                                                                                                                                          | Description                                      |
|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `_loanAccountDisbursementDetails` | `loanAccountDisbursementAccountNr`, `loanAccountDisbursementAmount`, `loanAccountDisbursementDate`, `loanAccountDisbursementKid` | Disbursement details to bank account            |
| `_panteobjekter`                | `pantPanteobjektnr`                                                                                                                                             | Collateral/security object (property etc.)      |
| `_sikkerhetsdokumenter`         | `sikkerhetDokumenttype`, `sikkerhetDagboknr`, `sikkerhetTinglystDato`, `sikkerhetUtlopsdato`                                                                   | Registered security document (e.g. mortgage deed) |

---

## Lookup sequence (typical flow)

```
1. GET /api/clients/{clientKey}          → fetch client data
2. GET /api/loans/{loanKey}              → fetch loan details
        ↳ find productTypeKey
3. GET /api/loanproducts/{productKey}    → fetch product configuration
4. GET /api/loans/{loanKey}/schedule     → fetch payment schedule
```

Note: Steps 3 and 4 can be performed in parallel after step 2.
