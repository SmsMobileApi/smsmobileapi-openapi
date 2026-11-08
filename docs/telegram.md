# Telegram connection and REST API

Connect your own Telegram account to SMSMobileAPI. Your application can find contacts, send text, images or documents, inspect sending status and read new private messages. Each customer connects their own Telegram account by QR code; SMSMobileAPI does not supply a shared sender.

## Connect the account

1. Open [Telegram Connection](https://dashboard.smsmobileapi.com/telegram-connect/) and generate a QR code. Choose whether SMSMobileAPI should retrieve received private messages. This is **optional and off by default**.
2. In Telegram on your phone, open **Settings → Devices → Link Desktop Device** and scan the QR code. Complete any confirmation requested by Telegram.
3. Open [Telegram Contacts](https://dashboard.smsmobileapi.com/telegram-contacts/) and synchronize contacts, or add a public `@username` manually.
4. Get the API key from [My API Key](https://dashboard.smsmobileapi.com/my-api-key/) and keep it on your server. Telegram endpoints can be enabled or disabled separately there.

You can change message reception at any time on the connection page. Turning it off stops new imports. Messages received during the pause are **not** imported later. Messages saved earlier remain in the [received history](https://dashboard.smsmobileapi.com/telegram-received/) until you clear it. Contact synchronization and sending remain available when reception is off.

## Authentication and base URL

Use `https://api.smsmobileapi.com` and send `X-SMSMobileAPI-Key: YOUR_API_KEY` in the HTTPS header. Unlike several SMSMobileAPI endpoints, these Telegram endpoints do not use the `apikey` query parameter. Keep the key out of public JavaScript, repositories, URLs and logs.

The examples below use placeholders only. A current SMSMobileAPI subscription and a connected Telegram account are required.

## Find a contact

```bash
curl -G 'https://api.smsmobileapi.com/telegram/contacts/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  --data-urlencode 'q=Alex' \
  --data-urlencode 'limit=50'
```

```json
{"ok":true,"data":[{"contact_id":42,"display_name":"Alex Example","username":"alex_example","source":"telegram"}],"paging":{"has_more":false,"next_cursor":null}}
```

`q` searches names or usernames. `limit` is 1–100, default 50. Pass `paging.next_cursor` as `cursor` to continue. Contact IDs belong to the connected SMSMobileAPI account.

## Send text

```bash
curl -X POST 'https://api.smsmobileapi.com/telegram/send/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"contact_id":42,"message":"Your appointment is confirmed.","idempotency_key":"2d40bb10-cc42-4c30-a282-93fc6dd8e161"}'
```

`message` allows 1–4096 characters, up to 16,000 UTF-8 bytes. The response uses HTTP 202 with `status: "queued"` and a `message_id`; this is **not** a delivery receipt. Generate one lowercase UUID-style `idempotency_key` for each intended send. Reuse the same key **and payload** when retrying after a timeout. An exact retry returns HTTP 200 and `duplicate: true`; a changed payload with the same key returns HTTP 409.

## Send an image or document

Use `multipart/form-data` with one `file`. Let your HTTP client set the multipart boundary:

```bash
curl -X POST 'https://api.smsmobileapi.com/telegram/send/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  -F 'contact_id=42' \
  -F 'message=Here is the document.' \
  -F 'file=@./document.pdf' \
  -F 'idempotency_key=66ab13f1-f451-4077-afdd-a7fdcaab3481'
```

JSON clients may instead provide `filename` and `file_base64` (raw Base64 or a data URI) with `contact_id`, an optional `message` caption and an optional `idempotency_key`:

```json
{"contact_id":42,"filename":"photo.png","file_base64":"BASE64_ENCODED_FILE","message":"See attached","idempotency_key":"cfcd44f1-4488-4519-95ac-5f7e04b52496"}
```

Supported files: JPG, PNG, WebP, PDF, TXT, CSV, JSON, ZIP, DOCX, XLSX and PPTX, up to **5 MB**. A caption can contain up to 1024 characters. Use multipart upload when possible to avoid Base64 overhead. The response is queued just like a text send.

## Read received private messages

Message reception must first be enabled on the [connection page](https://dashboard.smsmobileapi.com/telegram-connect/).

```bash
curl -G 'https://api.smsmobileapi.com/telegram/received/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  --data-urlencode 'limit=50'
```

```json
{"ok":true,"data":[{"message_id":8,"telegram_peer_id":123456,"telegram_message_id":82,"telegram_sender_id":123456,"sender_name":"Alex Example","username":"alex_example","message":"Hello!","type":"text","media":null,"received_at":"2026-11-08T10:00:00+00:00","captured_at":"2026-11-08T10:00:05+00:00"}],"paging":{"has_more":false,"next_cursor":null}}
```

The newest entries come first. Use `telegram_peer_id` to filter one private chat, `limit` from 1 to 100, and `cursor` / `paging.next_cursor` for older entries. Self messages, groups, channels and older history are excluded. For a received image or file, `type`, caption and file metadata are returned; **the received file bytes and a download URL are not available** in this release.

## Track sent messages and download sent files

```bash
curl -G 'https://api.smsmobileapi.com/telegram/sent/' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  --data-urlencode 'message_id=123'
```

The response lists `status` (`queued`, `sending`, `sent`, `error` or `unknown`), attempts, Telegram message ID, timestamps, `type` (`text`, `image` or `file`) and `media` metadata when an attachment exists. Optional filters are `contact_id`, `status`, `limit` (1–100) and `cursor`. `sent` means Telegram accepted the send; recipient delivery and read receipts are not reported. Do not blindly retry an `unknown` outcome.

Sent attachments remain in private storage for up to 90 days. The sent log provides a signed `media.download_url` valid for 15 minutes. Request the log again for a fresh link, or use the authenticated endpoint:

```bash
curl -L 'https://api.smsmobileapi.com/telegram/media/?message_id=123' \
  -H 'X-SMSMobileAPI-Key: YOUR_API_KEY' \
  -o attachment
```

The media endpoint serves **sent** attachments only. A signed download URL can be opened without the API key while it remains valid; treat it as a secret.

## Limits and errors

- One recipient per request; up to 15 new messages per minute and 30 queued messages per connected account. Respect Telegram rules and recipient consent.
- HTTP 401: missing or invalid key. HTTP 402: subscription expired. HTTP 403: endpoint disabled. HTTP 404: contact or file not found.
- HTTP 409: account not connected or idempotency conflict. HTTP 413/415/422: invalid size, type or input. HTTP 429: rate or queue limit. HTTP 503: temporary service or storage issue.

See the [interactive Telegram API guide](https://smsmobileapi.com/doc-telegram/), the [Telegram overview](https://smsmobileapi.com/telegram/) and the [OpenAPI contract](../openapi.yaml).

