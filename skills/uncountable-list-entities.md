---
name: uncountable-list-entities
description: >-
  Read rows out of the Uncountable R&D platform through the External API's list_entities
  endpoint, using a Listing Configuration defined in the Uncountable UI. Use this when you need
  experiment, recipe, project, ingredient or output data from Uncountable and you already know
  which Listing Configuration holds it.
api: uncountable-external-api
operations:
  - external_list_entities
generated: '2026-09-01'
method: generated
source: https://www.support.uncountable.com/knowledge-base/external-api-list_entities/
---

# Reading data out of Uncountable

## What is actually true about this API

Uncountable's External API does **not** let you query entities directly. There is no
`GET /experiments?project=…`. The query is defined in the product, by a human, as a
**Listing Configuration** — a saved base dataset plus the columns to return plus its filters —
and your call names that configuration. This is the single most important thing to understand
before writing any integration against Uncountable, and it is why a request that looks
under-specified is in fact complete.

The consequence: **if the data you want is not in a Listing Configuration, no API call will get
it.** Someone with UI access must create or amend the configuration first. Do not attempt to
work around this by constructing filters the configuration does not declare.

## Before you call

1. **Get a token.** OAuth 2.0 authorization code with PKCE (`S256`).
   - Authorize: `https://app.uncountable.com/oauth2/authorize`
   - Token: `https://app.uncountable.com/oauth2/token`
   - Scope: `EXTERNAL_API_READ` for reads.
   - The OAuth client is created by an administrator under **User Administration → OAuth**, and
     is scoped to a single schema/account. There is no dynamic client registration.
   - Basic authentication is documented as a legacy alternative, using a personal API key or a
     robot (service) user.
2. **Know your region.** The US deployment is `app.uncountable.com`; the EU deployment is
   `appeu.uncountable.com`. Tokens and Listing Configurations do not cross regions. A
   single-tenant customer has their own host.
3. **Get the `configReference`** of the Listing Configuration from whoever administers the
   Uncountable instance, along with the `entityType` it is built on.

## The call

Endpoint: `/api/external/entity/external_list_entities`

The request carries a top-level `data` object:

| Field | Required | Meaning |
|---|---|---|
| `entityType` | yes | the base entity type of the listing |
| `configReference` | yes | reference of the Listing Configuration defined in the UI |
| `limit` | no | maximum rows to return — **capped at 100** |
| `offset` | no | starting row index |
| `attributes` | no | values for the attribute-based filters the configuration declares |

The response mirrors the listing table as it appears in the UI: column definitions and row
values, in a consistent, ordered structure.

## Paging

`limit` is hard-capped at 100. Anything larger is a multi-request job: hold `configReference`
and `attributes` constant, advance `offset` by the page size, and stop when a page returns fewer
rows than you asked for. Uncountable states that rate limits exist but does not publish the
numbers or any `RateLimit-*` / `Retry-After` headers, so **you cannot read your remaining budget
off a response**. Pace conservatively and back off on any non-2xx rather than retrying tightly.

## Handling failure

- **401** — your token is missing, expired or wrong. The `WWW-Authenticate` header names the
  scopes required. Re-run the OAuth flow; do not retry the same token.
- **403** — you are authenticated but not entitled. Uncountable enforces the same role-based
  access control on the API as in the UI, at project, experiment and dataset level, and filters
  responses server-side. A 403 means someone must grant the user or robot user access — it does
  not mean the resource is missing, and retrying will not help. All access attempts are logged.
- **302 to `/signin`** — treat this as an authentication failure, not a success. Most
  unauthenticated application paths redirect to the sign-in page with an HTML body rather than
  returning a 401, so a client that only checks for a 2xx will happily parse a login page as
  data. Check the final URL, not just the status.

Empty results are a normal answer. A Listing Configuration whose filters match nothing returns
no rows, and that is data, not an error.

## Before you write anything

This skill covers reads only, and that is deliberate. Uncountable's authorization server
declares an `EXTERNAL_API_WRITE` scope, so a write surface exists — but its operations are
documented only behind the application sign-in at `https://app.uncountable.com/docs`, and
**Uncountable publishes no idempotency guarantee, no dry-run mode, and no reversal operation or
undo window on any public page**. Do not assume a write can be retried safely, and do not assume
it can be taken back. Establish both from the gated reference, or from Uncountable directly,
before granting an agent write scope on a production R&D dataset.

## Reference

- list_entities: <https://www.support.uncountable.com/knowledge-base/external-api-list_entities/>
- API access and permissions: <https://www.support.uncountable.com/knowledge-base/api-access-and-permissions-in-uncountable/>
- Full endpoint reference (requires sign-in): <https://app.uncountable.com/docs>
