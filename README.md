# SMSMobileAPI OpenAPI specification

Machine-readable API contract for building reliable integrations with SMSMobileAPI.

SMSMobileAPI connects a real mobile phone—with its SIM and existing number—to software through APIs, dashboards, integrations and Webhook V2 events. This repository makes the public HTTP contract easier to explore, validate and turn into client code.

## What is covered

- Send and retrieve SMS.
- Read sent-message logs and connected mobile devices.
- Retrieve missed, incoming and outgoing call activity.
- Send and list mobile notifications.
- Send WhatsApp messages through `/sendsms/` with `waonly=yes`.
- Activate and request consent-based WhatsApp synchronization.
- Retrieve synchronized WhatsApp messages.
- Send and retrieve e-mail through configured mailboxes.
- Understand the common Webhook V2 event envelope.

The authoritative human-readable documentation remains available at [smsmobileapi.com](https://smsmobileapi.com/documentations-api-smsmobileapi/).

## Files

| File | Purpose |
| --- | --- |
| [`openapi.yaml`](openapi.yaml) | OpenAPI 3.1 API and webhook contract. |
| [`examples/variables.env`](examples/variables.env) | Non-secret environment variable template. |

## Preview locally

Use any OpenAPI 3.1-compatible viewer, for example Swagger UI, Redocly or Stoplight.

```bash
npx @redocly/cli lint openapi.yaml
npx @redocly/cli preview-docs openapi.yaml
```

## Generate a client

The specification can be used with OpenAPI Generator. Review generated code before production use and keep API keys outside source control.

```bash
openapi-generator-cli generate \
  -i openapi.yaml \
  -g php \
  -o generated/php
```

Replace `php` with a supported generator such as `python`, `typescript-fetch`, `java` or `csharp`.

## Authentication

Most endpoints use the account API key in the `apikey` query parameter or form field. The specification models it as `ApiKeyQuery`.

- Store the API key in a protected server-side secret.
- Do not embed it in public JavaScript or mobile binaries.
- Do not commit a populated environment file.
- Mask credentials in logs and support messages.

## Important behavioral notes

- A successful send response means the platform accepted the job; mobile processing happens asynchronously.
- Prefer `POST` for messages, e-mail bodies and special characters.
- Use international phone-number format.
- WhatsApp retrieval requires activation and an explicit synchronization request before `/getwa/`.
- Webhook V2 uses at-least-once delivery; deduplicate the stable event ID.
- API capabilities depend on account configuration, subscription, connected devices and mobile permissions.

## Related repositories

- [API examples](https://github.com/SmsMobileApi/smsmobileapi-api-examples)
- [Postman collection](https://github.com/SmsMobileApi/smsmobileapi-postman)
- [Webhook examples](https://github.com/SmsMobileApi/smsmobileapi-webhook-examples)

## Contributing

Open an issue when the specification differs from the documented production behavior. Include the endpoint and a sanitized example, never credentials or customer data.

Released under the MIT License.
