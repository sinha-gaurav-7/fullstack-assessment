# StackShop — Stackline Full Stack Assignment

## Getting Started

```bash
yarn install
yarn dev
```

---

## Bugs Fixed

---

### Bug 1 — App Crashes on Listing and Detail Pages

**Files:** `app/page.tsx` · `app/product/page.tsx` · `next.config.ts`

#### Problems Identified

* Some products in the dataset have missing `imageUrls`. Accessing `product.imageUrls[0]` directly with no guard crashed the entire product grid. One bad product took down the whole listing page.
* The detail page had no validation when reading the product from the URL. A missing param, malformed JSON, or wrong data shape all resulted in a silent blank screen.
* A second Amazon CDN hostname (`images-na.ssl-images-amazon.com`) was missing from `next.config.ts`, causing a runtime crash on any product that used that image domain.

#### Changes Made

* Added optional chaining `imageUrls?.[0]` wherever the field is accessed and marked it optional in the TypeScript interface. TypeScript will now catch similar issues at compile time.
* Rewrote the detail page `useEffect` with layered guards. Missing param, malformed JSON, and wrong shape are each handled explicitly, with every failure path rendering the existing **"Product not found"** UI.
* Added the missing hostname to `remotePatterns` in `next.config.ts`.

#### Why This Approach

Optional chaining is the most concise inline guard with no restructuring of the JSX needed. Marking the interface optional is more honest than defaulting to `[]` because it forces TypeScript to flag every future access without a guard across the whole codebase. Layered guards were chosen over a single catch-all because each failure mode is a different problem and each deserves explicit handling, not one silent fallback.

---

### Bug 2 — Subcategory Filter Broken on Category Change

**File:** `app/page.tsx`

#### Problems Identified

* The subcategories fetch never passed `?category=` to the API. Every category selection returned subcategories from all categories mixed together. The API already supported this filter, it just wasn't being used.
* When switching categories, the subcategory dropdown showed stale options from the previous selection or disappeared entirely. `setSelectedSubCategory(undefined)` was firing in `onValueChange` at the same time as the `useEffect`, and the two were racing each other.

#### Changes Made

* Added `?category=` to the subcategories fetch so only relevant subcategories are shown.
* Moved all state resets into the `useEffect` and removed them from `onValueChange`. React now batches the resets in a single cycle before the new fetch fires, eliminating the race condition.

#### Why This Approach

The API was already built correctly so no backend changes were needed, just wiring up what was already there. Consolidating resets into the `useEffect` rather than splitting them across the handler and the effect is the correct React pattern. It makes state transitions predictable and ensures all related updates happen together in one render cycle.

---

### Bug 3 — Full Product Object Passed Through the URL

**Files:** `app/page.tsx` · `app/product/page.tsx`

#### Problems Identified

* Clicking a product serialized the entire object as JSON into the URL. All product data was exposed in plain text in the browser history and address bar.
* Products with long feature lists hit browser URL length limits, silently truncating the JSON and breaking `JSON.parse`.
* The listing page `Product` interface only had 5 fields. `featureBullets` and `retailerSku` were never included in the URL, so the **Features section and SKU on every detail page were always empty**.
* A fully working `/api/products/:sku` route already existed and returned the complete product. It just wasn't being used.

#### Changes Made

* Updated the product card link to pass only `stacklineSku` in the URL: `/product?sku=9PVEG4Y9`
* Rewrote the detail page `useEffect` to fetch the full product from the existing API using the SKU.

#### Why This Approach

Passing only the SKU fixes all three problems at the root rather than patching each symptom separately. The SKU is the right thing to put in a URL. It is short, safe, bookmarkable, and shareable. Fixing the JSON approach would still leave the security exposure and the data mismatch unresolved. The existing API already returned the full product with all fields, so using it was the obvious and correct solution.

---

## UX Improvements Suggested

These are not bugs but noticeable UX issues observed while using the app.

---

### UX 1 — No Pagination on Product Listing Page

The API supports `limit` and `offset` parameters but the listing page always fetches with a fixed `limit=20` and never loads more. If the dataset has more than 20 products, the user has no way to see them. There is no "Load More" button, no page numbers, and no indication that more products exist.

**Suggestion:** Add pagination controls below the product grid using the `total` count already returned by `/api/products`. A simple Previous / Next button with a current page indicator would make the experience complete.

---

### UX 2 — Feature Card Height Inconsistent Across Products

As seen in the product detail page, the Features card grows and shrinks depending on how many bullet points a product has. Products with a single feature get a very small card, while products with many features push the card very tall. This creates an inconsistent and unpolished layout.

**Suggestion:** Set a fixed max height on the Features card and make the content scrollable when it overflows. This gives every product detail page the same visual structure regardless of how much content exists.

```tsx
<CardContent className="pt-6 max-h-72 overflow-y-auto">
```

---

## Summary

| # | Issue | Files |
|---|-------|-------|
| 1a | `imageUrls` undefined crash | `app/page.tsx`, `app/product/page.tsx` |
| 1b | No validation on product param | `app/product/page.tsx` |
| 1c | Missing Amazon CDN hostname | `next.config.ts` |
| 2a | Subcategory showing all options | `app/page.tsx` |
| 2b | Stale subcategory on category switch | `app/page.tsx` |
| 3 | Full product JSON in URL | `app/page.tsx`, `app/product/page.tsx` |
| UX 1 | No pagination on listing page | `app/page.tsx` |
| UX 2 | Inconsistent feature card height | `app/product/page.tsx` |
