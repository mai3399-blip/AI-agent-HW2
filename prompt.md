# Dana's Bakery — LangGraph Order Processing System

## 1. Objective

Build a deterministic bakery order-processing system using Python and LangGraph in a Google Colab notebook.

The system must process customer orders, validate inputs, check inventory, handle substitutions, calculate discounts and delivery costs, generate customer messages, and maintain a chronological processing log.

The implementation must use standard Python logic only.

**Do not use LLMs, external AI APIs, API keys, or any AI-generated messages.**

## 2. Input

Each order contains exactly one product type and four input fields:

- `item`: the requested product.
- `quantity`: the requested number of units.
- `day`: the day of the week.
- `city`: the customer's city. An empty city means pickup.

Example invocation:

```python
result = graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem"
})
```

The graph must accept these four fields without requiring the caller to provide internal State fields.

## 3. Business Rules

### 3.1 Products, prices, and initial inventory

| Product | Price (NIS) | Initial inventory |
|---|---:|---:|
| challah | 18 | 10 |
| babka | 45 | 0 |
| rugelach | 30 | 3 |
| sourdough | 25 | 4 |

Rugelach refers to one box of rugelach.

Sourdough refers to one sourdough loaf.

Croissants were discontinued and are not valid products, even though they appeared on an old website.

Only the four active products above may be ordered.

Use the inventory values above as fixed initial values for evaluating each independent test order. Do not unintentionally carry inventory changes from one test invocation to another.

### 3.2 Input validation

Reject an order when:
- The product is not in the active product list.
- The quantity is zero or negative.
- The quantity is not a positive integer.
- Required product, quantity, or day information is missing or invalid.

Do not accept Python boolean values as quantities.

Handle invalid inputs gracefully without crashing.

### 3.3 Inventory and substitutions

An order may be accepted only if a single product has sufficient inventory to fulfill the entire requested quantity.

If the current product has insufficient inventory, check the next product in this substitution chain:

`babka → rugelach → sourdough → challah`

This chain represents the order of possible alternatives. It does NOT mean that every product must always be checked.

Rules:
- Keep the originally requested quantity.
- Never split an order across different products.
- Never accept a partially fulfilled order.
- Use the price of the product actually sold.
- Check at most TWO alternative products per order.
- Increment `suggestion_count` whenever an alternative is selected for stock checking.
- If the second alternative has insufficient inventory, reject the order.
- If there is no next product in the chain, reject the order.
- The substitution loop must always terminate.

Examples:

Example A:
A customer orders 5 boxes of rugelach.
- Rugelach: 3 available — insufficient.
- Sourdough: 4 available — insufficient, first alternative.
- Challah: 10 available — sufficient, second alternative.
Result: accept the order for 5 challahs.

Example B:
A customer orders 2 babkas.
- Babka: 0 available — insufficient.
- Rugelach: 3 available — sufficient, first alternative.
Result: accept the order for 2 boxes of rugelach.

Example C:
A customer orders 5 babkas.
- Babka: insufficient.
- Rugelach: insufficient, first alternative.
- Sourdough: insufficient, second alternative.
Result: reject the order. Do NOT continue to challah.

Example D:
A customer orders 11 challahs.
- Challah: insufficient.
- No next alternative exists.
Result: reject the order.

### 3.4 Discounts

Apply a 15% discount to the baked-goods subtotal only when BOTH conditions are true:

1. The subtotal is strictly greater than 150 NIS.
2. The order day is not Friday.

No discount applies on Friday.

No discount applies when the subtotal is exactly 150 NIS.

Delivery fees are never discounted.

Ignore the outdated 10% discount rule.

Use the following calculation:

`discount = subtotal * 0.15` when eligible, otherwise `0`.

### 3.5 Delivery and pickup

Delivery is available only in Jerusalem.

- If `city` is `"Jerusalem"`: set `delivery = "delivery"` and charge 20 NIS.
- If `city` is empty: set `delivery = "pickup"` and charge no delivery fee.
- If `city` is another city: automatically set `delivery = "pickup"` and charge no delivery fee.

For a nonempty city outside Jerusalem, set `delivery_changed = True`.

For Jerusalem or an empty city, set `delivery_changed = False`.

When delivery is automatically changed to pickup, clearly inform the customer.

The final total for an accepted order is:

`total = subtotal - discount + delivery_fee`

The delivery fee may be calculated locally in `check_delivery`; do not add `delivery_fee` as another State field.

### 3.6 Customer messages

Generate warm, clear, deterministic messages using Python templates.

Do not use an LLM.

Messages must explain relevant information, including:
- Successful order confirmation.
- The actual product and quantity.
- Product substitution, when applicable.
- Delivery or pickup.
- Automatic change from delivery to pickup, when applicable.
- Final price.
- Rejection reason, when applicable.

### 3.7 Processing log

Maintain `steps` as a chronological list of processing actions.

Every relevant Node should append meaningful entries.

The log must show:
- Input validation.
- Stock checks.
- Each alternative checked.
- Pricing and discount decisions.
- Delivery or pickup decisions.
- Rejection decisions.
- Final processing steps.

Use clear and consistent log entries.

## 4. LangGraph State

Define a typed LangGraph State containing exactly the following 14 fields.

| Field | Type | Description |
|---|---|---|
| `item` | str | Current product; may change after substitution |
| `quantity` | int | Requested quantity |
| `day` | str | Day of the week |
| `city` | str | Customer city |
| `original_item` | str | Originally requested product |
| `suggestion_count` | int | Number of alternative products checked, maximum 2 |
| `delivery_changed` | bool | Whether delivery was automatically changed to pickup |
| `subtotal` | float | Baked-goods subtotal before discount and delivery |
| `discount` | float | Discount amount in NIS |
| `status` | str | `"accepted"` or `"rejected"` |
| `delivery` | str | `"pickup"` or `"delivery"` |
| `total` | float | Final amount including delivery |
| `message` | str | Customer-facing message |
| `steps` | list[str] | Chronological processing log |

Do not add a `rejection_reason` field.

Do not add a `delivery_fee` field.

Initialize all internal fields safely so the graph can be invoked with only the four input fields.

Use lowercase `"pickup"` and `"delivery"` consistently.

The State may contain internal fields in addition to the six required output fields. This is acceptable.

## 5. LangGraph Nodes

Implement exactly these eight named Nodes.

### 5.1 `validate_order`

- Validate the requested product.
- Validate the quantity.
- Validate the required day information.
- Preserve the original requested product in `original_item`.
- Record validation results in `steps`.
- Route valid orders to `check_stock`.
- Route invalid orders to `reject_order`.

### 5.2 `check_stock`

- Check whether the current `item` has sufficient inventory for the requested `quantity`.
- Record the checked product, requested quantity, and available inventory in `steps`.
- If sufficient, route to `calculate_price`.
- If insufficient and another alternative is permitted, route to `find_alternative`.
- Otherwise, route to `reject_order`.

Use conditional edges for these routing decisions.

### 5.3 `find_alternative`

- Select the next product in the defined substitution chain.
- Update `item` to that product.
- Increment `suggestion_count`.
- Record the alternative selection in `steps`.
- Return to `check_stock` to verify the new product's inventory.

Never select more than two alternatives.

### 5.4 `calculate_price`

- Calculate `subtotal` using the current product's price and the requested quantity.
- Apply the correct discount rule.
- Update `subtotal` and `discount`.
- Record the calculation in `steps`.
- Continue to `check_delivery`.

### 5.5 `check_delivery`

- Determine whether the order uses delivery or pickup.
- Update `delivery`.
- Update `delivery_changed`.
- Calculate the final `total`, including any delivery fee.
- Record the delivery decision and fee in `steps`.
- Set `status = "accepted"` after successful stock verification, pricing, and delivery processing.
- Continue to `generate_message`.

### 5.6 `generate_message`

- Generate a deterministic customer message.
- Use the existing `status` to distinguish accepted orders from rejected orders, and the existing State and chronological `steps` to determine the appropriate message.
- Explain substitutions and delivery changes when applicable.
- Generate a clear apology when an order is rejected.
- Do not use an LLM.
- Continue to `finalize_order`.

### 5.7 `finalize_order`

- Ensure that all six required output fields are present and consistent.
- Verify and preserve `status = "accepted"` for successfully processed orders.
- Verify and preserve `status = "rejected"` for rejected orders; never change a rejected order to accepted.
- Ensure `total = 0` for rejected orders.
- Ensure the processing log is chronologically ordered.
- Continue to `END`.

### 5.8 `reject_order`

- Set `status = "rejected"`.
- Set `total = 0`.
- Record the specific reason for rejection in `steps`.
- Do not add a separate `rejection_reason` State field.
- Continue directly to `generate_message`.

## 6. Rejection Handling Without a Separate State Field

The State must NOT include `rejection_reason`.

Instead, record structured, consistent rejection information in `steps`.

For example:

- `REJECT:INVALID_ITEM`
- `REJECT:INVALID_QUANTITY`
- `REJECT:INVALID_DAY`
- `REJECT:INSUFFICIENT_STOCK`
- `REJECT:NO_ALTERNATIVE`

The `reject_order` Node must determine and record the correct reason using the existing State and previous processing steps.

The `generate_message` Node must use the recorded rejection information to produce the appropriate customer-facing message.

Do not display technical rejection codes directly to the customer.

## 7. Graph Structure and Conditional Edges

Implement a real LangGraph `StateGraph` with these paths:

```text
START
  |
  v
validate_order
  |-- invalid --> reject_order
  |
  |-- valid --> check_stock
                  |
                  |-- sufficient stock --> calculate_price
                  |
                  |-- insufficient stock -->
                          |
                          |-- another alternative allowed
                          |        |
                          |        v
                          |   find_alternative
                          |        |
                          |        v
                          |   check_stock
                          |
                          |-- no alternative allowed
                                   |
                                   v
                              reject_order

calculate_price
  |
  v
check_delivery
  |
  v
generate_message
  |
  v
finalize_order
  |
  v
END

reject_order
  |
  v
generate_message
  |
  v
finalize_order
  |
  v
END
```

Requirements:
- Use actual conditional edges in LangGraph.
- Reuse the same `check_stock` Node during the substitution loop.
- Do not create a separate stock-check Node for each product.
- Ensure the loop terminates after at most two alternatives.
- Rejected orders must bypass price calculation and delivery processing.
- Both accepted and rejected orders must reach `generate_message` and `finalize_order`.

## 8. Required Output

The graph must return these six fields:

1. `status`
2. `item`
3. `delivery`
4. `total`
5. `message`
6. `steps`

The output may also contain internal State fields.

For accepted orders:
- `status = "accepted"`
- `item` must identify the product actually sold.
- `delivery` must be `"pickup"` or `"delivery"`.
- `total` must include the correct discount and delivery fee.
- `message` must explain the order outcome.

For rejected orders:
- `status = "rejected"`
- `total = 0`
- `item` must retain the last product being evaluated, or the original invalid product if validation failed.
- `message` must clearly explain the rejection.
- `steps` must document the rejection.

For rejected orders that never reached delivery processing, use `delivery = "pickup"` as a deterministic default value.

## 9. Notebook Requirements

Create a Google Colab-compatible Jupyter notebook named:

`danas_bakery.ipynb`

The notebook must:

1. Install the required Python dependencies.
2. Import LangGraph and the necessary Python libraries.
3. Define the typed State.
4. Implement all eight Nodes.
5. Define the conditional routing functions.
6. Build and compile the LangGraph graph.
7. Display the graph using `draw_mermaid_png()`.
8. Include clear Markdown explanations.
9. Use the required test orders as executable examples; do not execute additional standalone example orders.
10. Execute and display exactly the ten original required test scenarios in Section 10.
11. Include tests that collectively cover all defined graph paths, including rejection.
12. Run successfully in a fresh Google Colab session without API keys.

## 10. Required Test Coverage

Execute and display exactly the following ten scenarios:

| Test | Order | Expected behavior |
|---|---|---|
| 1 | 2 challahs, Thursday, Jerusalem | Accepted, delivery, total 56 NIS |
| 2 | 2 babkas, Thursday, Jerusalem | Substitute with rugelach, accepted, total 80 NIS |
| 3 | 5 rugelach, Thursday, empty city | Substitute through sourdough to challah, accepted, total 90 NIS |
| 4 | 5 babkas, Thursday, Jerusalem | Rejected after two unsuccessful alternatives |
| 5 | 1 croissant, Monday, Jerusalem | Rejected as invalid product |
| 6 | 0 challahs, Monday, Jerusalem | Rejected as invalid quantity |
| 7 | 9 challahs, Thursday, Jerusalem | Accepted, 15% discount, total 157.70 NIS |
| 8 | 9 challahs, Friday, Jerusalem | Accepted, no discount, total 182 NIS |
| 9 | 2 challahs, Monday, Tel Aviv | Accepted, automatically changed to pickup, total 36 NIS |
| 10 | 2 challahs, Monday, empty city | Accepted, pickup, total 36 NIS |

Keep only these ten scenarios in the final notebook. Do not execute or display additional test scenarios or direct Node tests. Preserve all ten scenarios and their assertions and expected results.

Use assertions to verify expected results, not only printed output.

### 10.1 Readable test results

For each of the ten required test scenarios, display readable output in Google Colab using simple formatting:

1. Test number, description, and input order.
2. Actual output: `status`, final `item`, `delivery`, and `total`.
3. The customer-facing `message`.
4. The full chronological `steps` log, with each step on a separate line.
5. Expected versus actual values for the assertions, showing concrete compared values where applicable rather than only whether the assertion condition is true.
6. A clear PASS or FAIL indicator for each assertion and the overall scenario.

Preserve all ten required scenarios, their assertions, and expected results. Do not suppress failures or present failed tests as passing.

## 11. Technical Constraints

- Python implementation.
- LangGraph `StateGraph`.
- Deterministic business logic.
- No LLMs.
- No API keys.
- No external AI services.
- No persistent database required.
- No unnecessary external dependencies.
- Readable and well-structured code.
- Correct handling of invalid inputs.
- No infinite loops.
- No shared mutable State that accidentally contaminates separate test orders.

## 12. Deliverables

The final project must include:

- `prompt.md` — the technical specification and, if revised later, a documented list of changes.
- `danas_bakery.ipynb` — the complete Colab-compatible notebook, including graph visualization and test cases.

The project will eventually be submitted through a public GitHub repository, with an accessible Google Colab notebook link.

Do not create the GitHub repository until explicitly instructed.

## 13. Implementation Workflow

Follow this workflow:

1. Save this complete specification to `prompt.md`.
2. Wait for explicit approval before implementing.
3. After approval, create the notebook and implement the graph.
4. Run tests and verify all requirements.
5. If revisions are needed, document changes to the prompt.
6. Prepare the final deliverables for GitHub and Colab.

Do not proceed beyond step 1 in the current request.

## 14. Revision History

Before implementation, two clarifications were made to the original specification:

- Section 9: Removed the undefined cancellation requirement. Tests must cover all defined graph paths, including rejection.
- Section 5: Clarified that `check_delivery` sets `status = "accepted"` before message generation, `generate_message` uses the existing status, `reject_order` sets `status = "rejected"`, and `finalize_order` verifies and preserves the correct status.

These revisions did not change the State fields, Nodes, graph structure, business rules, or ten required test scenarios.


During development, the following presentation requirement was added:

- Section 10.1: Required readable results for every test scenario, including the input, actual output, customer message, chronological steps on separate lines, expected versus actual assertion values, and PASS/FAIL indicators. This change was made so reviewers can understand each order outcome and verify the tests without interpreting terse pass messages or reading the test code. Direct Node tests identify fields that do not apply. No business rules, State fields, Nodes, graph paths, assertions, expected results, or required scenarios were changed.

- Final test-scope simplification: Limited execution and reporting to the ten original Section 10 scenarios, retaining their assertions and expected values. Removed additional validation, branch, boundary, and isolation tests from the notebook and used required scenario 2 as the example instead of a duplicate standalone invocation. Added numbered descriptions while retaining detailed reports. This change reduces output to the requested ten scenarios without changing business rules, State fields, Nodes, graph routing, or required expected results. This supersedes the earlier references to additional and direct Node tests in the reporting revision.
