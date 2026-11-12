---
name: argip-api
description: Use when building, reviewing or debugging an integration with the Argip B2B Client API (argipapi.argip.com.pl) - products, stock, prices, orders, tokens from identityserver.argip.com.pl.
---

# Argip API integration

Read before writing code: https://github.com/argip/ArgipApi.AgentGuide/blob/master/AGENTS.md
(mandatory rules) and the matching file in
https://github.com/argip/ArgipApi.AgentGuide/tree/master/docs

Pick the example in the target language as a starting point:
https://github.com/argip/ArgipApi.AgentGuide/blob/master/examples.md

## Checklist before delivering code
- [ ] Token cached, refreshed 5 min before expiry, one retry after 401
- [ ] Secrets from environment/secret store, not in code or logs
- [ ] Quantities sent in pieces; prices read as per 100 pcs
- [ ] Order/Calculate before Order/Save; warnings shown to the user
- [ ] Order/Save result checked (Items, WarningMessages) - 200 does not mean saved
- [ ] Lists paged with pagesize <= 1000 using X-Pagination
- [ ] Empty 404 handled as a possible server error (log; retry reads only, never resend Order/Save or other writes)
- [ ] Max 500 items per order; `Content-Type: application/json`; lowercase `v1`
- [ ] CustomerOrderNumber max 30 characters; quantity 0 or less never sent (orders one full package)
- [ ] ExecutionDate = today or later in Europe/Warsaw (`yyyy-MM-ddT00:00:00`, no offset)
- [ ] No `null` in write bodies (use `""`); `Email` and `ReceivingCompanyName` in `PUT ShippingAddress` and `Items` in `Order/Save` are required
- [ ] Field lengths respected (addresses, customer parts, `PUT Products`, `PUT CustomTag`; see the guide)
- [ ] `PUT ShippingAddress` creates a real address: call it only with a testing-phase account, where it is validated but not saved
- [ ] Tested with TestingPhase = true before going live; there the writes (tags, mapping, customer parts, addresses) return `Message` and are not persisted, so skip read-back checks
