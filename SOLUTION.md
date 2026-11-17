# Solution

Tracking setup for the Arcvolt Pro X1 product page: Google Tag Manager (GTM) sends
page views, add-to-cart, checkout and newsletter sign-ups to Google Analytics 4 (GA4).

## 1. Setup

| What | ID |
|---|---|
| GTM container | `GTM-NPMZ3FKJ` |
| GA4 property (web stream "Arcvolt") | `G-X2TPP2DZH5` |

- The GTM snippet is in `index.html` in two places: the `<script>` in `<head>` and the
  `<noscript>` iframe right after `<body>`.
- The GTM setup is exported in `gtm-container.json`. It can be imported via
  GTM → Admin → Import Container.
- Screenshots from GTM Preview / Tag Assistant are in `screenshots/`.

**How to test it yourself:** GTM Preview doesn't work if you just double-click `index.html`.
The page has to be opened through a web address. The easiest way is a small local web
server:

1. Open a terminal in this folder.
2. Run `python -m http.server 8000`.
3. In GTM, click **Preview** and enter `http://localhost:8000/index.html`.

## 2. Data layer schema

The page sends these messages to the data layer. GTM reads them and forwards them to GA4.

| Event | When it fires | Data sent |
|---|---|---|
| `page_view` | Page loads | `page_title`, `page_location`, `page_type` (`"product"`) |
| `add_to_cart` | "Add to Cart" click | `ecommerce.currency`, `ecommerce.value`, `ecommerce.items[]` |
| `begin_checkout` | "Checkout" click | `ecommerce.currency`, `ecommerce.value`, `ecommerce.items[]` |
| `form_submit` | Newsletter form submitted | `form_name` (`"newsletter_signup"`), `form_location` (`"product_page_footer"`) |

Each ecommerce item has this shape:

```js
{
  item_id: 'ARC-PRO-X1',
  item_name: 'Arcvolt Pro X1',
  item_category: 'Performance Hardware',
  price: 1299,      // price of ONE item, as a number
  quantity: 3       // how many
}
```

`value` is the total (`price × quantity`), and `currency` is `"USD"`.

Event names use GA4's recommended names (`page_view`, `add_to_cart`, `begin_checkout`), so
GA4's built-in ecommerce reports work without extra setup.

## 3. GTM setup (tags, triggers, variables)

GTM works with three building blocks:

- **Tag:** what to send to GA4.
- **Trigger:** when to send it.
- **Variable:** which field to read from the page's data.

**Naming.** Every item starts with a short prefix, so it's clear what it is at a glance:
`GA4 -` for tags, `CE -` for triggers (Custom Event), `DLV -` for variables (Data Layer
Variable).

### Tags

| Tag | What it does | Fires when |
|---|---|---|
| `GA4 - Google Tag` | Connects the page to GA4. Runs first, on every page. | The page starts loading |
| `GA4 - Event - page_view` | Tells GA4 someone opened the page, and that it's a product page (`page_type`). | The page sends `page_view` |
| `GA4 - Event - add_to_cart` | Tells GA4 someone added to cart, with the product, price, quantity and total. | The page sends `add_to_cart` |
| `GA4 - Event - begin_checkout` | Same as above, for the checkout click. | The page sends `begin_checkout` |
| `GA4 - Event - form_submit` | Tells GA4 someone signed up, and which form it was (`form_name`, `form_location`). | The page sends `form_submit` |

For the two ecommerce tags, the option **"Send Ecommerce data"** is turned on. GTM then picks
up the product details from the page automatically, so they don't need to be mapped one by
one.

### Triggers

One trigger per event (`CE - page_view`, `CE - add_to_cart`, `CE - begin_checkout`,
`CE - form_submit`). Each one waits for the page to send that exact event name. The name must
match letter for letter, or the tag never fires.

### Variables

`DLV - page_type`, `DLV - form_name` and `DLV - form_location`. Each one reads a single field
from the page's data, so the tags can pass it on to GA4.

### Custom dimensions in GA4

GA4 receives `page_type`, `form_name` and `form_location`, but doesn't show them in reports
until they're registered. All three are registered as event-scoped custom dimensions
(GA4 → Admin → Custom definitions), so marketing can filter and break down reports by them,
e.g. sign-ups per form location (`screenshots/ga4-custom-dimensions.png`). GA4 only fills
them from the moment they're created, so they're set up before any real traffic.

### Why the automatic page view is turned off

- The Google tag sends a page view **automatically** on every page.
- The page **also** sends its own page view, which includes `page_type`.
- With both on, **every visit would be counted twice**: 100 visitors would show as 200.
- So the automatic one is turned off (`send_page_view = false` in the Google tag), and only
  the page's own page view is sent. GA4 gets exactly one page view per visit, including
  `page_type`.

Checked in GTM Preview: nothing is sent when the page starts loading, and one `page_view`
with `page_type = product` is sent after
(`screenshots/before-auto-page_view.png` → `screenshots/after-page_view.png`).

*Note for testing:* in GTM Preview, this page view appears in the event list as
"History Change". That is only how the debug tool labels it. Clicking it shows it is the
page's `page_view` event.

## 4. Bugs found and fixed

Both bugs were in the `add_to_cart` push. I found them by comparing it with
`begin_checkout`, which builds the same product data correctly.

### Bug 1: `items` misspelled as `itmes`

**What was wrong:** the product list was sent as `itmes` instead of `items`.

**Why it matters:** GA4 only reads product data from a field named exactly `items`. In Tag
Assistant, GA4 received a meaningless custom parameter `ep.itmes = [object Object]` and **no
product at all**. GA4 knew that something worth $1,299 was added to the cart, but not
*what*. Every product-level report (most-added products, cart-to-checkout rate per product,
item revenue) would be empty or wrong for add-to-cart.

**Fix and verification:** renamed to `items`. After the fix the hit contains a proper
product row (`pr1 = nmArcvolt Pro X1 ~ idARC-PRO-X1 ~ caPerformance Hardware ~ pr1299 ~ qt3`).
Before: `screenshots/before-add_to_cart.png`. After: `screenshots/after-add_to_cart.png`.

### Bug 2: `price` was text, and held the total instead of the unit price

**What was wrong:** the code was `price: String(PRICE * qty)`. That line has two problems:

1. `String(...)` turns the number into text (`'1299'` instead of `1299`). GA4 expects a number.
2. `PRICE * qty` is the **total**, not the price of one item. With quantity 3 it sent
   `price: '3897'`.

The comment in the original code only mentions the first problem. The second one only shows
up when quantity is more than 1. With quantity 1, total and unit price are the same number.

**Why it matters:** GA4 calculates item revenue as `price × quantity`. With quantity 3, it
would calculate 3,897 × 3 = **11,691** instead of the real **3,897**, overstating revenue 3×.
Text prices can also be dropped or misread, so revenue and price reports can't be trusted.

**Fix and verification:** changed to `price: PRICE` (1299, a number), the same as
`begin_checkout`. `value` stays `PRICE * qty` because it is the total. Tested with quantity 3:
the hit shows `pr1299~qt3` and `value = 3897`.

## 5. Other issues noticed

These were not part of the two marked bugs. I documented them rather than changing them.

- **Newsletter form counts invalid emails.** The form only checks that the fields aren't
  empty. Submitting the email `abc` still fires `form_submit`, so the sign-up numbers
  include addresses that can never receive email. It should only fire after the email is
  validated (or after the server confirms the sign-up).
- **No cookie consent.** There's no consent banner, so GA4 would track EU visitors without
  permission (a GDPR risk).
- **Ecommerce data isn't cleared between pushes.** GTM merges data layer pushes, so leftover
  `ecommerce` data from one event can leak into the next. Google recommends pushing
  `{ ecommerce: null }` before each ecommerce event.
- **Ad blockers block GTM.** GTM Preview only connected in an Incognito window, because a
  browser extension blocked `googletagmanager.com`. Real visitors with ad blockers are
  invisible in GA4 too.

## 6. Assumptions

- Single product, single currency (USD), so `PRICE` and `currency` are fixed in the code as
  provided.
- The GA4 property and GTM container are my own test accounts, created for this task.
- The provided `index.html` contained an extra third-party script at the top of `<head>`
  that isn't part of the task. It looked like it was added by browser monitoring software on
  the machine the file was saved from. I removed it. Nothing else on the page depends on it.

## 7. With more time

- **Cookie consent:** add a consent banner and Google Consent Mode, so GA4 only tracks
  visitors after they accept.
- **Clear old ecommerce data:** push `{ ecommerce: null }` before each add-to-cart and
  checkout event, so product data from one event can't mix into the next.
- **Form validation:** only send `form_submit` when the email is valid, so fake sign-ups like
  `abc` aren't counted.
