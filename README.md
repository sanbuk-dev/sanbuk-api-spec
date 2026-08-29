# Sanbuk API Spec

The public contract for the Sanbuk Tracking API — both halves of it, advertisers and publishers — and the source every official SDK is checked against.

**[راهنمای فارسی →](README.fa.md)**

| | |
|---|---|
| [`spec/openapi.yaml`](spec/openapi.yaml) | OpenAPI 3.1 description of every public endpoint |
| [`postman/sanbuk-tracking.postman_collection.json`](postman/sanbuk-tracking.postman_collection.json) | Seven requests, in the order you should try them |
| [`postman/sanbuk-production.postman_environment.json`](postman/sanbuk-production.postman_environment.json) | Environment pointing at the production edge |

`spec/` and the collection are generated from the Sanbuk backend and synced automatically whenever the contract changes. Edits made here are overwritten — open an issue instead.

## Advertisers: two endpoints, and one rule

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

## Publishers: four calls, and four obligations

A publisher normally installs a ready-made SDK — [`@sanbuk/js`](https://github.com/sanbuk-dev/sanbuk-js) for the web, [`sanbuk-android`](https://github.com/sanbuk-dev/sanbuk-android) for apps, or one of the shells built on it. This half of the spec is for everyone else: a game engine, a framework, or a platform we do not ship a shell for.

```
GET  https://t.sanbuk.com/ad?placement=…   # what should I draw here?
GET  https://t.sanbuk.com/i/{code}?eid=…   # it was seen
GET  https://t.sanbuk.com/c/{code}         # it was clicked
POST https://api.sanbuk.com/api/v1/sdk/ping  # this app media is real
```

The serving reply **describes** an ad — headline, body, image, call-to-action wording, brand colour. It is never markup. Draw it however suits your app; nothing here renders anything for you, which is the point.

Everything the shipped SDKs do beyond those four calls comes down to four obligations. Skipping them costs somebody money:

1. **Count a view only when it was seen** — at least half the ad on screen for a continuous second. Reporting on render counts ads nobody scrolled to.
2. **Send an `eid` with every impression, and reuse it when you retry.** The count has no event identity of its own, so a retried report without one is a second view — and a per-impression commission is paid on that number.
3. **Open the click in a real browser**, never an in-app WebView. A WebView has its own cookie jar, the first-party click id does not survive it, and the conversion is never attributed: the publisher does the work and earns nothing.
4. **Label the ad.** A reader has to be able to tell it from your own content.

Media integrated directly are marked as such and watched a little more closely, for the plain reason that we cannot verify the first two from here.

If the reply ever comes back with `"kill": true`, stop asking until your integration is updated. It means the build identifying itself in `sdk` is one we no longer answer, and it is the only way we can retire code that is already on people's phones.

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
