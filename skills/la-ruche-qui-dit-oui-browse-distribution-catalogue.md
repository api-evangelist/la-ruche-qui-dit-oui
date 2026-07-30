---
name: la-ruche-qui-dit-oui-browse-distribution-catalogue
description: >-
  Read the products and offers on sale for a La Ruche qui dit Oui! / Food
  Assembly distribution, and interpret the quantity and price representations
  correctly. Read-only and unauthenticated.
api: la-ruche-qui-dit-oui:food-assembly-api
generated: '2026-07-19'
method: generated
source: openapi/la-ruche-qui-dit-oui-api-openapi.yml
operations:
  - listDistributionProducts
---

# Browse a distribution catalogue

Lists what a given distribution is selling. This is the only operation the
provider documents as public — no bearer token is required.

## Before you start

The provider's documentation was last updated in April 2017 and has drifted from
the live deployment. On 2026-07-19 this route returned `404` with the generic
`{"problemType":"/exception"}` envelope for distribution id `1`. Verify the
route resolves for your distribution id before relying on it, and treat a `404`
as "route or id not found", not as "distribution is empty".

## Steps

1. Obtain the distribution id you want to read. The API documents no operation
   that lists or searches distributions — you must already have the id.
2. Call `listDistributionProducts` — `GET /distribution/{id}/products/` on
   `https://api.thefoodassembly.com`. Note the singular `distribution` segment
   and the required trailing slash; the basket route uses the plural form.
   Send no `Authorization` header.
3. Read the response as a bare JSON array of products. There is no envelope,
   no `count`, and no pagination — you receive everything the distribution has.

## Interpreting the payload

Each product carries one or more `offers`, and the offer is the purchasable
unit. Decode both quantity and price before showing anything to a user:

- **Price** is integer minor units plus an ISO 4217 currency.
  `{"amount": 780, "currency": "EUR"}` is **EUR 7.80** — divide by 100 for EUR.
- **Quantity** is an integer in the base unit named by the offer.
  `{"amount": 2000000, "unit": "mg"}` is **2 kg**. The owning product's `type`
  object repeats the measurement contract as `quantityUnit` and
  `quantityStrategy` (for example `weight`).
- **`availableStock`** may be the string `unlimited` rather than a number.
  Handle both types; do not coerce blindly.
- **`storageLife`** is `{amount, unit}` where `amount` is a *string*
  (`{"amount": "2", "unit": "days"}`).
- **`farmId`** references the producer, but no farm endpoint is documented — you
  can group products by farm, not resolve farm details.
- `description`, `composition`, `freshness`, `count` and `photoId` are all
  nullable.

## Errors

The generic envelope is `{"problemType": "/exception", "title": "", "detail": []}`.
`title` and `detail` were empty in every observed response, so the body carries
no diagnostic value — rely on the status code. See
`errors/la-ruche-qui-dit-oui-problem-types.yml`.
