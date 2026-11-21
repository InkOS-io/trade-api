# Integrate with InkOS Trade

**Partner preview · draft v1.** InkOS Trade connects a print shop that needs work produced with a wholesale printer, sign maker or fabrication partner that can produce it. This guide and the [OpenAPI contract](openapi.yaml) are intended to be publicly readable and downloadable before a company creates an account. The API, credentials and sandbox described here are planned. This preview has no live service address.

## Choose your starting point

| If you are integrating… | Your first milestone |
| --- | --- |
| A trade printer's production system | Publish one service with its limits and trade price. Receive a matching purchase order, acknowledge it and send back shipment and delivery events. |
| A retail print shop, broker or its software | Describe a job once, find a supplier who can make it, obtain a firm offer and dispatch a purchase order. Use the trade order ID to follow the job in your own system. |

The supplier controls which buyers see each trade rate. The public documentation describes how matching and pricing work. A buyer sees the agreed price and freight on its offer. The supplier's production costs, the buyer's retail margin, customer history and unrelated orders stay in their respective workspaces.

## Your first trade order

Use this fictional order to plan a sandbox integration. All amounts are CAD. Amounts on the API are integer minor units, so 15000 represents $150.00.

| Step | Action | Expected result |
| --- | --- | --- |
| 1. Publish | The supplier publishes a versioned postcard capability covering the stock, size, duplex colour, quantity of 1,000, and blind shipping. It makes the price available to the test buyer. | An eligible buyer can discover the service. A different buyer outside the approved group cannot see the restricted rate. |
| 2. Price | The buyer requests an offer for 1,000 postcards and provides a destination and required date. | The dated offer identifies the supplier, applicable capability version, expiry, $150 print, $30 freight and $0 example tax. Check the promised date before committing to the customer. |
| 3. Dispatch | After retail acceptance and artwork release, the buyer submits its work order and $180 PO using the offer reference and a stable dispatch key. | The supplier receives one order with the released artwork revision, delivery instructions and packing slip. The buyer's $217.50 retail selling price is private. |
| 4. Produce and ship | The supplier reads its event feed, retrieves the order, acknowledges it and posts carrier and tracking details at dispatch. | The buyer can follow the same order in its own system. Delivery evidence closes fulfillment. If delivery fails, the broker owns the customer follow-up. |
| 5. Reconcile | The **trade printer invoices the buying shop and collects directly**. Each side matches the invoice to the trade order. | A $60 direct payment against $180 leaves $120 owed to the supplier. The buyer’s retail invoice and InkOS subscriptions are separate. The draft contract below covers production and delivery. |

## Try the proposed interface

The relative API base is `/api/trade/v1`. The [contract](openapi.yaml) contains the proposed operations, request fields, responses and authorization requirements. This first draft covers fixed-specification offers and a pollable event feed. Custom quotes, webhook registration and invoice synchronization are planned separately.

**Publish a service.** With a supplier catalog credential, send the following body to `PUT /capabilities/demo-postcards/versions/1`. It prices one run of 1,000 double-sided 5 × 7 in cards and grants the example buyer access. Substitute the actual test buyer ID when credentials become available. An identical repeat returns `204`. To change the price, publish version 2.

```json
{
  "product": "postcards",
  "widthMm": 127,
  "heightMm": 177.8,
  "media": "100 lb cover",
  "sides": 2,
  "colour": "full-colour",
  "finish": "trim",
  "quantity": 1000,
  "price": {"currency": "CAD", "printMinor": 15000, "freightMinor": 3000, "taxMinor": 0},
  "effectiveAt": "2026-11-21T00:00:00Z",
  "leadBusinessDays": 3,
  "shipCountries": ["CA"],
  "blindShippingSupported": true,
  "eligibleBuyerIds": ["demo-buyer"]
}
```

**Acknowledge a job.** Read `GET /events` with the supplier credential. A `dispatched` event gives you the `orderId`. Retrieve the packet with `GET /orders/{orderId}` and its released file with `GET /orders/{orderId}/artwork`. In this example, `INKOS_TRADE_BASE` is the full address of the future test environment including the base path. Set `ORDER_ID` to the ID from the feed.

```sh
curl --request POST "$INKOS_TRADE_BASE/orders/$ORDER_ID/events" \
  --header "Authorization: Bearer $SUPPLIER_TOKEN" \
  --header "Content-Type: application/json" \
  --data '{"eventId":"evt-ack-1","type":"acknowledged","occurredAt":"2026-11-21T16:00:00Z"}'
```

The buyer should see one acknowledgement for the order. After a lost response, resend the same event ID with the same content. For buyer dispatch, reuse the original `Idempotency-Key` and identical request. After a disconnected period, read the event feed from the last saved cursor, process each event once, and save the next cursor after processing succeeds.

## Signage and fabrication

A supplier can publish an exact run of ten CNC-cut, 600 × 300 mm acrylic sign blanks. Set `product` to `sign-blank`, `process` to `CNC routing`, `media` to `cast acrylic`, `materialThicknessMm` to `6`, `cutPathFormat` to `closed vector contours` and `finish` to `polished edges`. Omit press fields such as `sides` and `colour` for an unprinted job. Include the run quantity, price, eligibility, effective date, lead time and destination countries as in the postcard example. To quote a different thickness or finish, publish the matching capability version. When released to the supplier, the production packet refers to the exact approved artwork revision.

## Before sending production work

Use these checks to evaluate an integration. Obtain a test buyer, supplier and scoped credentials when sandbox enrollment opens.

| Test | Pass condition |
| --- | --- |
| Change a rate after issuing an offer | A new offer shows the new price. The original valid offer retains its terms until expiry. Dispatching an expired offer requires a new offer. |
| Lose a dispatch response and retry | Both attempts resolve to the same trade order. Changing the contents under the same key returns a conflict. |
| Upload new artwork after release | The supplier receives the exact revision the buyer released for that work order. |
| Deliver an event twice or late | Each event affects the order once. An older production update cannot reverse confirmed delivery. |
| Integrate as a second supplier | It can access its own jobs. An order ID from the first supplier grants it no access to that packet or its artwork. |

## Availability and changes

This draft is available for design review. The sandbox opening date, production enrollment steps, integration fees, numerical usage limits, support contact and service targets are undecided. Before enrollment opens, InkOS plans to publish those details alongside a dated changelog, instructions for obtaining test credentials, and a way to report integration problems. Breaking changes should have a documented migration path and retirement date. Keep the contract version with your implementation so you can compare revisions.

## Companion integration

A print business connecting its own estimating, customer service or production system can use the separate customer contract. Supplier discovery, trade prices and buyer-to-supplier dispatch are covered here. [InkOS customer API](https://github.com/InkOS-io/customer-api)
