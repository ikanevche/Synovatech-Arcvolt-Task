# Analytics Engineering — Take-Home Task

---

## Context

You're joining a team that relies heavily on accurate analytics data to make product and marketing decisions. This task simulates a real scenario: a product page exists, some tracking is already in place, but it has issues — and it needs to be extended.

---

## The Page

A static product page (`index.html`) for a fictional product called the **Arcvolt Pro X1**. It includes:

- A product detail section with an **Add to Cart** button
- A **Proceed to Checkout** button that reveals a checkout flow
- A **newsletter signup form** at the bottom

---

## Your Task

### 1. Connect GTM

- Create a free GTM container at [tagmanager.google.com](https://tagmanager.google.com)
- Replace `GTM-XXXXXXX` in `index.html` with your container ID (appears twice — in the `<head>` and the `<noscript>` tag)

### 2. Connect GA4

- Create a free GA4 property at [analytics.google.com](https://analytics.google.com)
- Set up a GA4 Configuration tag in GTM firing on **All Pages**

### 3. Implement event tracking via GTM

Using the existing `dataLayer` pushes already in the page, create GTM tags to send the following events to GA4:

| Event | Trigger |
|---|---|
| `page_view` | Page load |
| `add_to_cart` | Add to Cart button click |
| `begin_checkout` | Proceed to Checkout button click |
| `form_submit` | Newsletter form submission |

For `add_to_cart` and `begin_checkout`, pass through the ecommerce parameters (`value`, `currency`, `items` array) as GA4 event parameters.

### 4. Find and fix the bugs

There is **deliberately broken tracking code** in `index.html`. Find it, fix it, and document:

- What the bug(s) are
- Why they matter (what would break in GA4/reporting if left unfixed)
- How you fixed them

### 5. Document your work

Include a `SOLUTION.md` file in your fork with:

- Your data layer schema (what parameters each event sends)
- Your event naming rationale
- Bug findings and fixes
- Any assumptions you made
- What you would do differently or add given more time

---


## What We Evaluate

| Area | What we look for |
|---|---|
| **Correctness** | Tags fire on the right triggers with the right parameters |
| **Data quality** | Correct data types, no missing required fields, consistent naming |
| **Bug detection** | Did you find both bugs? Did you explain why they matter? |
| **Documentation** | Could a non-technical marketer understand your schema? |
| **Depth** | Did you go beyond the minimum? (consent handling, custom dimensions, validation approach) |

---

## Submission

1. Create a repo
2. Make your changes directly to `index.html`
3. Add a `SOLUTION.md` with your documentation
4. Send us the GitHub link

Good luck.
