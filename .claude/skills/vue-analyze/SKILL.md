---
name: vue-analyze
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Use when asked to audit, review, or optimize Vue components.
context: fork
agent: vue-expert
---

# Vue Component Analyzer

Analyze the Vue 3 component(s) specified in `$ARGUMENTS`. If no arguments are given, analyze all `.vue` files under `client/src/`.

## Your Task

1. **Read** each target component using the Read tool
2. **Analyze** each one against the checklist below
3. **Report** findings grouped by category, with file path and line number for each issue
4. **Suggest** concrete fixes — show before/after code snippets where helpful

---

## Analysis Checklist

### Reactivity & Computed Properties

- [ ] Is raw data stored in `ref`/`reactive` and derived data in `computed`? (Never recompute in the template)
- [ ] Are expensive computations memoized with `computed` instead of recalculated on every render?
- [ ] Are `watch`/`watchEffect` used appropriately — only for side effects, not for deriving state?
- [ ] Does any `computed` property have unnecessary `watch` duplicates?

**Pattern to flag:** Logic inside `<template>` that calls methods or filters inline (e.g., `{{ formatCurrency(item.price) }}` inside a `v-for` loop without memoization).

### v-for Optimization

- [ ] Does every `v-for` use a stable, unique `:key` (not array `index`)?
  - Prefer `sku`, `id`, `month`, or other domain identifiers
- [ ] Are `v-for` + `v-if` on the same element? (Move `v-if` to a wrapping `<template>` or filter with `computed`)
- [ ] Are large lists candidates for virtual scrolling?

### Component Decomposition & Reuse

- [ ] Are there repeated template blocks (>15 lines) that appear in 2+ files? → Extract to a shared component
- [ ] Are there large view files (>300 lines) doing too many things? → Identify sub-components to split out
- [ ] Do multiple components share identical `props` + `emits` signatures? → Consider a composable or base component
- [ ] Are inline SVG icons repeated? → Candidate for an `<AppIcon>` component

**Check for duplication across this project's views:**
- `client/src/views/Dashboard.vue`
- `client/src/views/Inventory.vue`
- `client/src/views/Orders.vue`
- `client/src/views/Spending.vue`
- `client/src/views/Backlog.vue`
- `client/src/views/Demand.vue`
- `client/src/views/Restocking.vue`
- `client/src/views/Reports.vue`

### Composable Extraction

- [ ] Is there stateful logic (data fetching, filter state, formatting) duplicated across multiple components? → Extract to `client/src/composables/use<Feature>.js`
- [ ] Does any component mix API call logic with rendering logic? → Separate with a composable
- [ ] Are there utility functions defined inside `<script setup>` that don't use reactive state? → Move to `client/src/utils/`

**Common composable candidates in this codebase:**
- Filter state management (warehouse, category, timePeriod, orderStatus)
- API fetch + loading/error state pattern
- Currency/date formatting helpers

### Props & Emits

- [ ] Are all props typed with `defineProps<{...}>()` or `defineProps({ prop: { type, required } })`?
- [ ] Are emits declared with `defineEmits`?
- [ ] Is prop drilling more than 2 levels deep? → Consider `provide/inject` or a shared composable

### Template Readability

- [ ] Are ternary expressions in templates longer than one condition? → Move to `computed`
- [ ] Are there magic numbers/strings in the template? → Extract to `const` in `<script setup>`
- [ ] Is `v-show` used where `v-if` is correct (or vice versa)? (`v-show` for frequent toggles, `v-if` for conditional existence)

### Performance

- [ ] Are event handlers inside `v-for` creating new closures every render? → Use a single handler with the item passed as argument
- [ ] Are images/heavy assets loaded eagerly when they could be lazy?
- [ ] Are child components imported statically when they could be `defineAsyncComponent`?
- [ ] Does any component re-fetch data on every mount that could be cached?

---

## Output Format

Structure your response as:

```
## [filename] — [line count] lines

### Critical (fix now)
- Line XX: [issue] → [fix]

### Suggested (worth doing)
- Line XX: [issue] → [fix]

### Reuse Opportunity
- [Description of what could be extracted and where it's duplicated]
```

End with a **Summary Table**:

| File | Critical | Suggested | Reuse Opportunities |
|------|----------|-----------|---------------------|
| Dashboard.vue | 2 | 3 | 1 |
| ...  | ... | ... | ... |

And a **Top 3 Highest-Impact Actions** section listing the changes that would give the most benefit across the whole codebase.
