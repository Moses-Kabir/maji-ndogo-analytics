# Maji Ndogo Farm Survey: Notebook Review

## 1. What breaks when the data grows to a million records, and what was built to fix it

Notebook 1 only benchmarks 5,654 records. At that size almost anything is instant, so the problem only shows up when you scale the numbers.

### What breaks

The cost of `linear_search` grows in step with the data. It checks records one by one, so a million records means about **177 times more work** than 5,654, on every lookup. Run it thousands of times and the system stalls.

### What was built to fix it

| Function | Complexity | Why it scales |
|---|---|---|
| `binary_search` | O(log n) | Halves the search range each step. A million records takes about 20 comparisons instead of up to a million. Needs sorted input, so you pay for one sort and then get cheap lookups. |
| `merge_sort` | O(n log n) | Ranks fields by yield. A naive O(n²) sort would take around 10¹² operations at a million records, while this one takes around 20 million. The trade-off is O(n) extra memory. |
| `filter_by_pollution` | O(n) | A single pass using `filter()` and a lambda. |
| `project_harvest` | O(n) | A single pass using `map()` and a lambda. |

`filter_by_pollution` and `project_harvest` can't be made faster than O(n), since every record must be looked at, but they never get worse than linear.

The point of the benchmark is to show the gap between O(n), O(log n) and O(n log n) as the data grows, not to make 5,654 records fast.

---

## 2. Why objects that check their own data make a system safer

In notebook 2, Sanaa's complaint is that a dictionary will accept `Pollution_level = 7000` without complaint. Nothing else in the system would notice either, because every function that touches the dictionary has to remember to check it.

The `Field` class fixes this with **encapsulation**:

- `pollution_level` is a `@property` backed by a private `_pollution_level`.
- The setter raises `ValueError('Pollution level must be between 0 and 1.')` for anything outside `[0, 1]`.
- The check also runs inside `__init__`, so an invalid object can never be created.

### Why this is safer

- **One place to enforce the rule.** Other engineers can trust that any `Field` they receive is valid, rather than re-checking everywhere.
- **Bad data fails at the source.** An impossible sensor reading raises an error when it's loaded, not three functions later as a wrong total.
- **Structure is enforced too.** `AbstractCropField` can't be instantiated until `crop_name()` is implemented, and `build_field` returns `None` for crops with no price. Both stop malformed or unpriced objects from entering the registry.

### Trade-off

Because `build_field` passes through the setter, a single corrupt record raises an error and halts the whole `FieldRegistry` build. In production you would probably catch the error per record, log it, and carry on.

---

## 3. Why the tea recommendation is based on evidence, not opinion

The closest thing is the pricing data in notebook 2:

| Crop | Price per ton |
|---|---|
| `TeaField` | $1,180 |
| `CoffeeField` | $2,400 |
| `WheatField` | $320 |

`FieldRegistry` computes `total_projected_yield()` and `total_projected_revenue()` across the 2,840 priced fields.

If the recommendation is built on those outputs, the evidence-versus-opinion argument runs like this:

- **It comes from computed figures.** Yield × price per ton, summed over survey records, not a judgement about which crop seems best.
- **The numbers can be checked.** They come from validated objects with polymorphic methods, and anyone can re-run them.
- **Bad readings can't distort the totals.** The validation layer stops an impossible sensor reading from quietly skewing the results.
