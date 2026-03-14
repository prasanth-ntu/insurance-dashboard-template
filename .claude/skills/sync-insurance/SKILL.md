---
name: sync-insurance
description: Sync the HTML dashboard policy data from the markdown overview. Use when Family Insurance Overview.md has been updated and the dashboard needs to reflect the changes. Triggers on "/sync-insurance" or when user says "sync insurance dashboard" or "update dashboard from markdown".
---

## Purpose

Regenerate the `const policies = [...]` data array in `Family Insurance Dashboard.html` from the source of truth in `Family Insurance Overview.md`.

## Steps

1. **Read** `Family Insurance Overview.md` (the source of truth)
2. **Read** `Family Insurance Dashboard.html` (the target)
3. **Extract** all policy data from the markdown tables for each family member section, mapping to this schema:

```
{
  person,       // Family member name (from section heading)
  age,          // Age (from section heading; 0 for infants, null if unknown)
  role,         // Role (Father, Mother, Daughter, Sam's Mother, etc.)
  insurer,      // Insurer name (from table or section heading)
  policyNo,     // Policy number (from table; null if "—")
  policy,       // Plan name (strip markdown bold ** and status annotations like *(proposed)*)
  type,         // Type column value
  category,     // Derived category (see mapping below)
  country,      // Country column value (SG or IN)
  status,       // Derived status (see rules below)
  premium,      // Annual premium as number in SGD (strip $, commas; convert INR at noted rate)
  cpfMA,        // CPF MA as number (0 if "—" or empty)
  cpfOA,        // CPF OA as number (0 if "—" or empty)
  cash,         // Cash as number (0 if "—" or empty)
  coverage,     // Key Coverage column value (plain text, strip markdown formatting)
  paymentTerm,  // Premium Payment Term column value
  period        // Cover Period column value
}
```

4. **Category mapping** (from `type` field):
   - "Integrated Shield" → `"IP"`
   - "IP Rider", "Worldwide Rider" → `"IP Rider"`
   - "CareShield Supp.", "CPF Board" where plan contains "CareShield" → `"CareShield"`
   - "CPF Board" or "CPF Board (GE)" where plan contains "DPS" → `"DPS"`
   - "CPF Board" where plan contains "HPS" → `"HPS"`
   - "Personal Accident", "PA Rider" → `"PA"`
   - "Term Life", "Corporate plan" → `"Term Life"`
   - "WL + CI" → `"CI/WL"`
   - "IP (Class A)" → `"IP"`

5. **Status derivation** (from plan name annotations and context):
   - `*(declined)*` or `~~*(proposed)*~~` → `"Declined"`
   - `*(proposed)*` → `"Proposed"`
   - `*(to buy)*` → `"To Buy"`
   - `*(deferred)*` → `"Deferred"`
   - "Employer" in insurer or type contains "Corporate" → `"Employer"`
   - No annotation and has a policy number or is CPF Board → `"Active"`

6. **Currency conversion**: For India (IN) policies, convert INR to SGD using the exchange rate noted in the markdown footnotes (e.g., "1 SGD = ₹72"). Extract the INR premium and convert.

7. **Special cases**:
   - Skip "Total" summary rows (plan name starts with "Total")
   - Skip "Term Life" rows marked "Not needed"
   - For bundled policies (premium shows "bundled above"), set premium to 0
   - For "Employer-paid" policies, set premium to 0
   - For TBD premiums, set premium to 0
   - Preserve `policyNo` as string (some have leading zeros)

8. **Replace** the data block in the HTML file between the markers:
   ```
   // ── POLICY DATA ──
   ```
   and
   ```
   // ── END POLICY DATA ──
   ```
   Keep the comment header (Source of truth, Schema lines) and update `// Last synced:` to today's date.

9. **Validate** after replacement:
   - Count of policies matches the number of non-Total, non-skipped rows across all tables
   - Sum of active premiums matches the "Total (active SG policies)" rows
   - Each person's CPF MA + CPF OA + Cash = Premium for active policies

10. **Report** to the user:
    - Number of policies synced per person
    - Any changes detected vs. previous data (new policies, removed policies, changed values)
    - Any warnings (missing data, ambiguous parsing, validation mismatches)

## Important Notes

- The markdown is the **sole source of truth**. Never modify the markdown — only read it.
- The `persons` array and `personColors` in the HTML should NOT be modified by this skill.
- Only the `const policies = [...]` block and the `Last synced` date should change.
- If a new family member appears in the markdown that isn't in the `persons` array, warn the user that they need to manually add the person to the dashboard config.
