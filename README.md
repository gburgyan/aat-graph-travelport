# Travelport+ Air v11 — AAT Graph & Templates

> **Disclaimer:** This project is **not affiliated with, endorsed by, or produced by Travelport**. It is solely the work of George Burgyan, built entirely from publicly available Travelport API documentation. No proprietary or confidential Travelport materials were used in its creation.

This directory contains a comprehensive AAT graph definition and adapter templates for the **Travelport+ JSON Air v11 API**. It models the full air booking lifecycle including search, pricing, reservation management, ticketing, exchanges, and ancillary services.

## Files

| File | Description |
|------|-------------|
| `graph.yaml` | API graph: 30 nodes, ~100 edges, element fields for array selection |
| `domain.yaml` | Domain knowledge: airport codes, passenger types, carriers, cabin classes, etc. |
| `templates/` | 30 HTTP adapter templates (one per graph node) |

## Environment

Use the existing environment at `environments/travelport-pp.yaml` which configures:
- Base URL for Travelport PP (pre-production)
- OAuth 2.0 Bearer token authentication
- Required headers: `XAUTH_TRAVELPORT_ACCESSGROUP`, `Accept-Version: 11`, `Content-Version: 11`

## Booking Flows

### Reference-Based Flow (recommended)

The standard 8-step GDS booking flow using identifier references between API calls:

```
searchFlights → priceOfferReference → createWorkbench → addOfferReference
→ addTraveler → addFormOfPaymentCash/Card → addPayment → commitReservation
```

### Full-Payload Flow

An alternative flow that sends complete trip details instead of references:

```
searchFlights → priceOfferFullPayload → createWorkbench → addOfferFullPayload
→ addTraveler → addFormOfPaymentCash/Card → addPayment → commitReservation
```

### Post-Commit Ticketing

For separate ticketing after a held booking:

```
commitReservation → createWorkbenchFromLocator → addFormOfPaymentCash/Card
→ addPayment → commitTicket
```

### Optional Flows

| Flow | Nodes |
|------|-------|
| Multi-leg search | `searchFlights → searchFlightsNextLeg` |
| Upsell search | `searchFlights → searchUpsells` |
| Fare rules | `getFareRulesFromSearch` or `getFareRulesFromOffer` |
| Ancillaries | `searchAncillaries` (in workbench context) |
| Seat map | `searchSeatMap` (in workbench context) |
| Ticket retrieval | `getTicketByLocator` |
| Ticket void | `getTicketByLocator → voidTicket` |
| Cancellation | `retrieveReservation → cancelOffer` |
| Exchange | `getExchangeEligibility → searchExchange` |
| Special services | `addSpecialService` (in workbench context) |
| Queue placement | `placeOnQueue` |

## Graph Nodes (30 total)

### Search (3)
- **searchFlights** — Initial flight search (CatalogProductOfferings)
- **searchFlightsNextLeg** — Multi-leg continuation (BuildNext)
- **searchUpsells** — Upsell options (BuildOptions)

### Pricing (2)
- **priceOfferReference** — Price by reference to search identifiers
- **priceOfferFullPayload** — Price with full product details

### Reservation Workbench (4)
- **createWorkbench** — New empty workbench
- **createWorkbenchFromLocator** — Workbench from existing PNR
- **ignoreWorkbench** — Discard workbench (cleanup)
- **retrieveWorkbench** — Get current workbench state

### Offers (2)
- **addOfferReference** — Add offer to workbench (reference)
- **addOfferFullPayload** — Add offer to workbench (full payload)

### Travelers (2)
- **addTraveler** — Add passenger with name, contact, documents
- **addPrimaryContact** — Add primary contact for the booking

### Payment (3)
- **addFormOfPaymentCash** — Cash FOP
- **addFormOfPaymentCard** — Credit card FOP
- **addPayment** — Apply payment to an offer

### Commit & Management (3)
- **commitReservation** — Commit workbench → PNR
- **retrieveReservation** — Retrieve by identifier or locator
- **cancelOffer** — Cancel an offer in a reservation

### Ticketing (3)
- **commitTicket** — Commit with ticket issuance
- **getTicketByLocator** — Retrieve tickets by PNR
- **voidTicket** — Void a ticket

### Ancillaries & Seats (2)
- **searchAncillaries** — Bags, meals, etc.
- **searchSeatMap** — Seat availability

### Fare Rules (2)
- **getFareRulesFromSearch** — Rules from search results
- **getFareRulesFromOffer** — Rules from priced offer

### Exchange (2)
- **getExchangeEligibility** — Check exchange eligibility
- **searchExchange** — Search for exchange options

### Other (2)
- **addSpecialService** — SSR requests (wheelchair, meals, etc.)
- **placeOnQueue** — Move reservation to agency queue

## Key Identifier Patterns

Travelport uses three identifier patterns:

| Pattern | Scope | Example |
|---------|-------|---------|
| `id` | Local, intra-message | `CatalogProductOffering.id` |
| `*Ref` fields | Cross-reference within a message | `productRef` |
| `Identifier.value` | Globally unique GUID across API calls | `Reservation.Identifier.value` |

The graph edges use `Identifier.value` for cross-node data flow since it persists across API calls.

## Remarks

- **Price/Offer request bodies**: The reference flow constructs pricing requests from `CatalogProductOfferingsIdentifier` + `CatalogProductOfferingSelection`. The `productIds` input is a placeholder for comma-separated product IDs selected from search results.
- **Search modifiers**: The searchFlights template uses a minimal request body. Optional inputs like `cabinPreference`, `carrierPreference`, and `maxConnections` require adding a `SearchModifiersAir` object (noted in template comments).
- **Return flights**: Round-trip searches need a second `SearchCriteriaFlight` entry with the return date. Alternatively, use `searchFlightsNextLeg` for the return leg.
- **Form of Payment**: Only Cash and PaymentCard are templated. Travelport supports 11+ FOP subtypes (BSP, Agency, Voucher, etc.).
- **Exchange flow**: The exchange search template is partially modeled. A full exchange workflow would also need exchange pricing and exchange commit steps.
- **NDC content**: Templates default to GDS content. NDC flows may require different request structures for some operations.
- **Workbench cleanup**: Nodes that create workbenches have `cleanup: ignoreWorkbench` to ensure workbenches are discarded if the test fails mid-flow.
