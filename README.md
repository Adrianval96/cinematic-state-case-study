# Building a production print shop, solo

**Next.js, Payload CMS, Stripe, and a webhook-driven fulfilment pipeline.**

I designed, built, and operate [cinematicstatephotography.com](https://cinematicstatephotography.com) — a photography portfolio with a print shop behind it. An order flows from Stripe checkout to a print-on-demand order at Printful, with confirmation and shipping emails on the way.

| | |
|---|---|
| **Stack** | Next.js App Router · Payload CMS 3 · Postgres (Neon) · Stripe Embedded Checkout · Printful · Resend · Vercel |
| **Process** | 384 integration tests · about 300 merged pull requests · staging and production pipelines · feature-flagged rollouts |
| **Status** | The site is live. The shop is built and sits behind a feature flag until the print collection is ready. |

The application repository is private, so this write-up carries the code. The excerpts below are taken from it as of November 2026, trimmed for length.

Four problems turned out to be more interesting than "I made a website with a shop."

## 1. Multi-currency pricing without lying to anyone

Prints are priced in AUD, but buyers are in Europe, the UK, Canada and the US. The naive options are bad: hardcode per-currency prices (stale the day you set them) or show only AUD (conversion anxiety kills checkout).

The model I landed on: **AUD is the single source of truth**. Every other price is derived from FX rates cached for 24 hours. If the rate service fails, fallback rates take over and a staleness flag travels with the order, into Stripe's session metadata and onto the order record, for forensics.

```ts
async function fetchRates(): Promise<CurrencyRateResult> {
  try {
    const response = await fetch(
      'https://api.frankfurter.app/latest?from=AUD&to=EUR,GBP,CAD,USD',
    )
    if (!response.ok) {
      return { rates: FALLBACK_RATES, isStale: true }
    }
    const data = await response.json()
    return { rates: data.rates as FxRates, isStale: false }
  } catch {
    return { rates: FALLBACK_RATES, isStale: true }
  }
}

export const getCurrencyRate = unstable_cache(fetchRates, ['currency-rate'], {
  revalidate: 86400,
})
```

Converted prices snap to retail-shaped endings — €48.37 becomes €48.99, never €48.00:

```ts
export function snapToValidPrice(rawPrice: number): number {
  return Math.floor(rawPrice) + 0.99
}
```

Geo-IP picks the default currency server-side; a switcher stores the buyer's explicit choice in the browser. Shipping went through the same inversion: zones used to carry fixed per-currency rates, which created a second conversion model, so I dropped the currency column by migration and made shipping AUD-base too. One conversion path for everything on the page.

The honest trade-off, made deliberately: non-AUD prices move with the FX rate. The alternative is a fixed price list per currency, which is five lists to maintain for a small catalog.

## 2. The cart that can't be tampered with

The cart stores **identifiers only** — variant ID, product slug, quantity and a timestamp. No prices. Display prices are resolved server-side through a single endpoint, and at checkout the server re-resolves the entire cart from the CMS and builds the Stripe line items itself. A buyer can edit localStorage all day; the price authority never leaves the server.

The subtle case: a product archived between "add to cart" and "pay." The cart marks that line "No longer available"; the checkout route catches the race with a generic `409` and sends the buyer back to the cart, which already owns that UX:

```ts
const { resolved, dropped } = await resolveCartItems(items)
if (dropped.length > 0) {
  return NextResponse.json({ error: 'cart_changed' }, { status: 409 })
}
```

An earlier design had a per-item dropped-list contract and a dedicated interstitial; I cut it as scope creep. The rare race didn't deserve its own UI surface.

## 3. Embedded checkout on a dark-theme site

Stripe Embedded Checkout renders white by default, and CSS can't reach inside the iframe. The only code-level theming path is per-session `branding_settings` — literal hex values at session creation. The site is warm ink `#0b0a09` with bone `#ece5d8` text, and the checkout page background and Pay button take those two values. The form panel itself stays white: `branding_settings` cannot colour it. Matching it fully means replacing the embed with a custom form styled through Stripe's Appearance API, which is not built yet.

More interesting: **when** the session gets created. The page originally minted a Stripe session on load — every visit created an orphan session and a pending order row. Now a country gate ("Ship to" selector) stands before the embed, because the session bakes in a single computed shipping rate and needs the destination *first*. The session is created only on the explicit "Proceed to payment" click, with the country locked so the buyer can't desync from the computed rate inside the iframe:

```ts
shipping_address_collection: {
  allowed_countries: [country],
},
shipping_options: [
  {
    shipping_rate_data: {
      type: 'fixed_amount',
      fixed_amount: { amount: quote.shipping, currency },
      display_name: SHIPPING_DISPLAY_NAME,
    },
  },
],
```

Visiting checkout costs nothing; proceeding mints exactly one session and one pending order.

## 4. Fulfilment as a webhook state machine

A client-side redirect is not proof of payment. The success page sends no email and mutates nothing. The authority is Stripe's signed `checkout.session.completed` webhook: verify the signature, find the order by its unique, indexed `stripeSessionId`, and check it's still `pending` so redeliveries are no-ops.

```ts
async function handleCheckoutCompleted(session: Stripe.Checkout.Session): Promise<void> {
  const order = await findOrderByStripeSessionId(session.id)
  if (!order) {
    console.error(`[Stripe webhook] No order found for session ${session.id}`)
    return
  }

  if (order.status !== ORDER_STATUS.PENDING) {
    return
  }
  // … mark paid, send the confirmation email, then fulfil
}
```

The amounts were computed on the server when the session was created and stored on the pending order then; the webhook flips it to `paid` and takes the payment reference, email and shipping address from Stripe's event, never from the browser.

Then fulfilment: create the Printful order; on success → `fulfilled`, on failure → `paid_unfulfilled` plus an ops alert. A Printful failure is caught and recorded, so a print-provider outage does not make Stripe retry forever. Printful orders are created as **drafts and confirmed by hand** for now: drafts are free and cancellable, and automatic confirmation waits for a hardening pass. Downstream, Printful's own signature-checked webhooks close the loop: `package_shipped` → tracking email to the buyer; `order_failed` → action-required alert, because an optimistic `fulfilled` that silently never ships is the lie this design exists to prevent.

## The process is the product

Solo doesn't mean cowboy. Every change: a named branch, a small focused pull request, integration tests, type-checked, linted, auto-deployed to staging on merge and to production on a version tag. Feature flags live in the CMS and gate anything unfinished, so production stayed boring while the shop was built behind them.

The stack is learnable. Shipping production software alone, safely, on a budget, is the skill.
