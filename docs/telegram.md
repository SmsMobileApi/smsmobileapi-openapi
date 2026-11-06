# Telegram connection and API

Connect your own Telegram account to SMSMobileAPI, synchronize its contacts, send a text message from that account, and inspect the sending log. Each SMSMobileAPI customer authorizes their own Telegram account; SMSMobileAPI does not provide a shared sender.

## Connect and send your first message

1. In the dashboard, open [Telegram → Connection](https://dashboard.smsmobileapi.com/telegram-connect/) and start the QR-code connection.
2. In your Telegram mobile app, open **Settings → Devices → Link Desktop Device** and scan the QR code. Complete any Telegram confirmation requested on your phone.
3. Open [Telegram → Contacts](https://dashboard.smsmobileapi.com/telegram-contacts/) and synchronize contacts. You can also add a saved `@username` there. Use the resulting `contact_id` for API sends.
4. Open [My API Key](https://dashboard.smsmobileapi.com/my-api-key/) and store your SMSMobileAPI API key in a protected server-side secret. The Telegram endpoints can be enabled or disabled individually there.
5. Send from [Telegram → Send a message](https://dashboard.smsmobileapi.com/telegram-send/) or through the API below. Check [Telegram → Sending log](https://dashboard.smsmobileapi.com/telegram-logs/) for the outcome.

One Telegram account can be connected to each SMSMobileAPI account. Disconnect it from the dashboard whenever needed.

## API authentication

Base URL: `https://api.smsmobileapi.com`

Send your account API key in the `X-SMSMobileAPI-Key` HTTPS request header for **all three Telegram endpoints**. Keep the key out of browser code, public repositories, screenshots, and logs. These endpoints use the header, while several other SMSMobileAPI endpoints use an `apikey` parameter.

### 1. Find a contact

```bash
curl -G 'https://api.smsmobileapi.com/telegram/contacts/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  --data-urlencode 'q=Alex' \
  --data-urlencode 'limit=50'
```

```json
{"ok":true,"data":[{"contact_id":42,"display_name":"Alex Example","username":"alex_example","source":"telegram"}],"paging":{"has_more":false,"next_cursor":null}}
```

The optional `q` searches display names and usernames. Use `limit` (1–100, default 50) and `cursor` / `paging.next_cursor` for pagination. A contact ID is scoped to the connected SMSMobileAPI account; it cannot be reused across customers. Synchronize contacts again when a new contact is missing.

### 2. Send a text message

```bash
curl -X POST 'https://api.smsmobileapi.com/telegram/send/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"contact_id":42,"message":"Your appointment is confirmed."}'
```

```json
{"ok":true,"message_id":123,"status":"queued","idempotency_key":"GENERATED_UUID_V4","duplicate":false}
```

`contact_id` and `message` are required. The text must contain 1–4096 characters and at most 16,000 UTF-8 bytes. For reliable retries, also provide an optional `idempotency_key`: generate a new lowercase UUID v4 for each intended message and reuse **the same key and payload** after a timeout. An exact retry returns HTTP 200, `duplicate: true`, and the original `message_id`; changing the payload with the same key returns HTTP 409.

HTTP 202 means **queued**, not sent or delivered. Save `message_id` and inspect the sending log.

### 3. Check the sending log

```bash
curl -G 'https://api.smsmobileapi.com/telegram/sent/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  --data-urlencode 'message_id=123'
```

```json
{"ok":true,"data":[{"message_id":123,"contact_id":42,"contact_name":"Alex Example","username":"alex_example","message":"Your appointment is confirmed.","status":"sent","telegram_message_id":781,"error_code":null,"error_detail":null,"attempts":1,"queued_at":"2026-11-06T09:42:00+00:00","sent_at":"2026-11-06T09:42:04+00:00"}],"paging":{"has_more":false,"next_cursor":null}}
```

Optional filters: `message_id`, `contact_id`, `status` (`queued`, `sending`, `sent`, `error`, `unknown`), `limit` (1–100), and `cursor`. Results are newest first. `sent` means Telegram accepted the message from the connected account. Recipient delivery and read receipts are not reported. If the state is `unknown`, the result could not be confirmed; do not blindly resend.

## Limits and common errors

- A request sends to **one synchronized contact** or saved username at a time. This is currently a text sending and sending-history integration; receiving Telegram messages is not yet exposed by these endpoints.
- The current limit is 15 new messages per minute and 30 queued messages per connected account. Telegram rules and recipient consent still apply.
- HTTP 401: missing or invalid API key. HTTP 403: endpoint disabled in My API Key. HTTP 404: contact not found in this account.
- HTTP 409: Telegram not connected or idempotency conflict. HTTP 422: invalid message or pagination. HTTP 429: queue or rate limit. HTTP 503: service temporarily unavailable.

See the [interactive Telegram API documentation](https://smsmobileapi.com/doc-telegram/), [Telegram overview](https://smsmobileapi.com/telegram/), and the [OpenAPI contract](../openapi.yaml) for the request and response schemas.
