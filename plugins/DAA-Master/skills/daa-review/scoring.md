# DAA Score & Report Format

## DAA Score

Every review report MUST start with a **DAA Score** (1-10). This gives humans an immediate, intuitive sense of the code's DAA compliance.

### Score Rubric

| Score | Grade | Meaning |
|-------|-------|---------|
| 10 | Perfect | Full DAA compliance — no violations found |
| 8-9 | Excellent | Minor SUGGESTION-level issues only |
| 6-7 | Good | WARNING-level issues but no CRITICAL violations |
| 4-5 | Needs Work | 1-2 CRITICAL violations or many WARNINGs |
| 2-3 | Poor | Multiple CRITICAL violations — DAA structure is largely broken |
| 1 | Failing | Code does not follow DAA at all — essentially flat procedural scripts |

### Scoring Rules

1. **Start at 10**, then deduct:
   - Each CRITICAL finding: **-2 points**
   - Each WARNING finding: **-1 point**
   - Each SUGGESTION finding: **-0.5 points** (round final score to nearest integer)
2. **Floor at 1** — score never goes below 1
3. **Cap deductions at category level**: Even with 10 SUGGESTIONs, the SUGGESTION category deducts at most 2 points total

## Report Structure

Use this structure for every review report:

```
## DAA Review Report

### DAA Score: 6 / 10

### Summary
- X CRITICAL findings
- Y WARNING findings
- Z SUGGESTION findings

### Findings

#### [CRITICAL] AP-2: Fire-and-Forget Action — missing self-verification
**File**: actions/cart_actions.py:15
**Code**: `add_to_cart(item_id)` executes click but never verifies the cart updated
**Fix**: Add assertion after click: `assert cart_count() == expected, "Item not added"`

#### [WARNING] AP-7: Inconsistent naming
**File**: actions/search_actions.py:42
**Code**: `do_search(keyword)` — missing verification suffix
**Fix**: Rename to `search_for_product_and_verify_results_not_empty(keyword)`
```
