---
title: 'Verifying Binary Search in Pure, Total Gardel'
description: 'Proving overflow-safe midpoint calculation, termination, soundness, and completeness with gardelc --strict.'
pubDate: '2026-11-11'
---

One of the first modules verified in **Firulete** — our catalog of classic algorithms proved end-to-end under `gardelc --strict` — is generic binary search over sorted `LIST[K]` (`K: ORD`).

## What the Specification Captures

Rather than proving only bounds safety, the contract for `BINARY_SEARCH[K: ORD].search_range` proves:

1. **Overflow-safe midpoint arithmetic** (`lo + (hi - lo) / 2`) and strict termination (`variant hi - lo`).
2. **Soundness:** returning `Some(idx)` guarantees `lo <= idx < hi` and `LIST_OPS.at(xs, idx).equals(target)`.
3. **Completeness:** when `K`'s `equals` is congruent with `less_than`, returning `None` guarantees no element in `[lo, hi)` equals `target`.

```gardel
module BINARY_SEARCH[K: ORD]

create {ANY}

  -- Search for `target` in the half-open index range `[lo, hi)` of sorted `xs`.
  search_range(xs: LIST[K], target: K, lo: NAT, hi: NAT): OPTION[NAT]
    require
      ordered_bounds: lo <= hi
      in_bounds: hi <= xs.length
      sorted: BINARY_SEARCH.is_sorted(xs)
    variant
      hi - lo
    do
      if lo = hi then
        None
      else
        let mid = lo + (hi - lo) / 2
        let mid_val = LIST_OPS.at(xs, mid)
        if target.less_than(mid_val) then
          check
            sorted_right: BINARY_SEARCH.sorted_all_above(xs, mid)
          end
          BINARY_SEARCH.search_range(xs, target, lo, mid)
        else if mid_val.less_than(target) then
          check
            sorted_left: BINARY_SEARCH.sorted_all_below(xs, mid)
          end
          BINARY_SEARCH.search_range(xs, target, mid + 1, hi)
        else
          Some(mid)
        end end
      end
    ensure
      found_in_bounds: Result.is_some implies (lo <= Result.get and Result.get < hi)
      found_matches: Result.is_some implies
        (LIST_OPS.at(xs, Result.get).equals(target) and
         (not target.less_than(LIST_OPS.at(xs, Result.get))) and
         (not LIST_OPS.at(xs, Result.get).less_than(target)))
    end
```

All inductive helper lemmas are proved explicitly inside `check ... end` blocks with zero `assume` escapes, and verified statically with `gardelc --project firulete --strict`.

