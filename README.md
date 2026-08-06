# Sanbuk API Spec

The public contract for the Sanbuk Tracking API, and the source every official SDK is checked against.

**[راهنمای فارسی →](README.fa.md)**

| | |
|---|---|
| [`spec/openapi.yaml`](spec/openapi.yaml) | OpenAPI 3.1 description of both public endpoints |
| [`postman/sanbuk-tracking.postman_collection.json`](postman/sanbuk-tracking.postman_collection.json) | Seven requests, in the order you should try them |
| [`postman/sanbuk-production.postman_environment.json`](postman/sanbuk-production.postman_environment.json) | Environment pointing at the production edge |

`spec/` and the collection are generated from the Sanbuk backend and synced automatically whenever the contract changes. Edits made here are overwritten — open an issue instead.

## Two endpoints, and one rule

```
POST https://t.sanbuk.com/v1/postback   # server to server — billable
POST https://t.sanbuk.com/v1/pixel      # browser — verification only
```

**Postback is the financial source of truth.** Only a postback can create a payable conversion.

**Pixel is verification and anti-fraud.** It never bills on its own.

Send the *same* `event_id` on both. Sanbuk pairs them inside a 48-hour window, and that pairing is the only thing the integration asks you to get right.

## Integration checklist

1. **Keep `snbk_cid`.** Sanbuk adds it to your landing URL. Store it next to the order or user it belongs to. Without it a conversion is recorded but can never be attributed to a publisher, and never paid.
2. **Pick an `event_id` you already own and never reuse.** An order id is ideal. A repeat answers `200` and changes nothing, so retrying is always safe.
3. **Rehearse with `X-Sanbuk-Mode: test`** until the action reads "verified" in the panel. A test event verifies your setup but never spends your wallet, and is a separate event from its live twin — so rehearsing with a real order id never consumes it.
4. **Drop the header to go live.**

All monetary values are integers in **Rial**.

## Trying it

Import the collection and the production environment into Postman, fill in `apiKey` and `pixelId` from the advertiser panel, and run the requests top to bottom. They are ordered deliberately: sandbox first, then live, then a deliberate duplicate, then the two error cases.

```bash
curl -X POST https://t.sanbuk.com/v1/postback \
  -H "Authorization: Bearer $SANBUK_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Sanbuk-Mode: test" \
  -d '{
        "action": "purchase",
        "event_id": "ORD-10293",
        "click_id": "018f3c2e-7b1a-7c3d-9e4f-1a2b3c4d5e6f",
        "value": 25000000
      }'
```

`202` means the event is new. `200` means it was already seen and nothing changed. Both are success.

## Errors

Branch on `code`, never on `message` — the message is localised and may change.

| Code | HTTP | Meaning |
|---|---|---|
| `invalid_api_key` | 401 | The Bearer key is missing or wrong |
| `invalid_pixel_id` | 404 | No workspace owns that pixel id |
| `unknown_action` | 404 | The action code is not defined in your workspace |
| `value_required` | 422 | This action prices on value, so `value` is mandatory |
| `validation_failed` | 422 | The body is malformed; `errors` names the fields |

`429` means rate limited. Wait and send the same `event_id` again — that is what idempotency is for.

## Official SDKs

You do not have to call the API directly.

| Language | Package | Repository |
|---|---|---|
| JavaScript / TypeScript | `@sanbuk/js` | [sanbuk-js](https://github.com/sanbuk-dev/sanbuk-js) |
| PHP | `sanbuk/sanbuk-php` | [sanbuk-php](https://github.com/sanbuk-dev/sanbuk-php) |
| Python | `sanbuk` | [sanbuk-python](https://github.com/sanbuk-dev/sanbuk-python) |

Each one handles idempotency, retries with backoff, and the click id for you.

## Versioning

`info.version` in the spec tracks the contract, not the SDKs. Breaking changes get a new path prefix; everything else is additive.

## License

Apache-2.0
