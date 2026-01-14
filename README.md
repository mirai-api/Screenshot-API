## Overview

Screenshot API lets you generate high-quality screenshots of web pages (or specific elements on a page) via a simple two-step workflow:

1) **Create a task** with `POST /create`  
2) **Fetch the result** with `GET /result?id=...`

It supports:
- Desktop / phone / tablet rendering
- Full-page screenshots (with safe height limits)
- Element screenshots (single element or multiple matches stacked vertically)
- Removing intrusive UI (cookie banners, popups) via CSS selectors
- Optional waits and delays for dynamic content
- Output formats: **JPG**, **PNG**, **WebP**
- Custom headers (cookies, language, etc.)

## Demo

Try the live demo:
https://mirai-api.github.io/Screenshot-API/

Tip: open your browser DevTools → **Network** tab to see the exact requests and responses used by the demo.

---

## Authentication

All requests go through RapidAPI.  
You can get your `X-RapidAPI-Key` here:
https://rapidapi.com/apimirai/api/screenshot-api8/

In most HTTP clients you don’t have to set headers manually, but if you do, send:

- `X-RapidAPI-Key: {RAPIDAPI_KEY}`
- `X-RapidAPI-Host: screenshot-api8.p.rapidapi.com`

---

## Quick Start

### 1) Create a screenshot task

```bash
curl -s -X POST "https://screenshot-api8.p.rapidapi.com/create" \
  -H "Content-Type: application/json" \
  -H "X-RapidAPI-Key: {RAPIDAPI_KEY}" \
  -H "X-RapidAPI-Host: screenshot-api8.p.rapidapi.com" \
  -d '{
    "url":"https://example.com",
    "device":"desktop",
    "format":"webp",
    "full_page":true
  }'
````

Response (success):

```json
{
  "status": "accepted",
  "task_id": "9f0d2b5b4c3a1e..."
}
```

### 2) Poll for the result

```bash
curl -s "https://screenshot-api8.p.rapidapi.com/result?id=9f0d2b5b4c3a1e..." \
  -H "X-RapidAPI-Key: {RAPIDAPI_KEY}" \
  -H "X-RapidAPI-Host: screenshot-api8.p.rapidapi.com"
```

While running:

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "pending"
}
```

When ready:

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "done",
  "content_type": "image/webp",
  "image_base64": "UklGRiQAAABXRUJQVlA4..."
}
```

On failure:

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "error",
  "error": "Timeout while loading the page"
}
```

---

## Endpoint: `POST /create`

Create a screenshot task.

### Request

* **Method:** `POST`
* **Content-Type:** `application/json`
* **Body:** JSON object described below

### Response

Always JSON:

* `status: "accepted"` + `task_id`
* or `status: "error"` + `error`

---

### Parameters (JSON body)

#### Required

* `url` (string)
  Target page URL. Only `http://`/`https://` are allowed.

#### Optional — Rendering & viewport

* `device` (string)
  One of: `desktop`, `phone`, `tablet`.
  If unknown/empty, defaults to `desktop`.

* `width` (number|string)
  Viewport width. Can be a number (`1366`) or a string (`"1366"`).
  If omitted, a device-based default is used.

* `height` (number|string)
  Viewport height. Can be number or string.
  If omitted, a device-based default is used.

* `zoom` (number)
  Page zoom multiplier (e.g. `1.0`, `1.25`, `0.9`). Default: `1.0`.

* `full_page` (boolean)
  If `true`, captures the full page (with safety height limits). Default: `false`.

#### Optional — Element screenshots

* `element_selector` (string)
  CSS selector for element screenshot.

  * If it matches **one** element → returns that element screenshot
  * If it matches **multiple** elements → screenshots each match and **stacks them vertically** (top to bottom)
  * If it matches **none** → returns an error
    You may pass multiple selectors separated by commas, for example: `".card,#main,.hero"`

* `element_wait_state` (string)
  How the element should appear before screenshot:
  `visible` (default), `attached`, `hidden`, `detached`

* `selector_timeout_ms` (number)
  How long to wait for the element to reach the desired state.
  Range: `0..30000` ms (values above are clamped).

#### Optional — Waiting for dynamic content

* `wait_for_selector` (string)
  Wait for a selector before taking the screenshot (page or element flow).

* `wait_for_state` (string)
  State to wait for: `visible` (default), `attached`, `hidden`, `detached`

* `wait_for_timeout_ms` (number)
  How long to wait for `wait_for_selector`.
  Range: `0..30000` ms (values above are clamped).

* `delay_ms` (number)
  Extra delay after load/waits (useful for animations and lazy-loaded blocks).
  Range: `0..30000` ms.

#### Optional — Removing overlays/popups

* `remove_selectors` (string)
  Comma-separated CSS selector list to remove from the page before screenshot.
  Example: `".cookie-banner,#subscribe-modal,header.sticky,img"`
  Invalid selectors are ignored (best-effort).

#### Optional — Output format

* `format` (string)
  One of: `jpg`, `jpeg`, `png`, `webp`
  Default: `jpg`

* `quality` (number)
  Used only for `jpg` and `webp`. Range: `1..100`.
  Defaults: `jpg=85`, `webp=80`. Ignored for `png`.

#### Optional — Custom headers

* `headers` (object)
  Extra HTTP headers to send when loading the page.
  Example:

  ```json
  {
    "headers": {
      "cookie": "session=abc",
      "accept-language": "en-US,en;q=0.9"
    }
  }
  ```

---

### Example: Element screenshot + cleanup + WebP

```bash
curl -s -X POST "https://screenshot-api8.p.rapidapi.com/create" \
  -H "Content-Type: application/json" \
  -H "X-RapidAPI-Key: {RAPIDAPI_KEY}" \
  -H "X-RapidAPI-Host: screenshot-api8.p.rapidapi.com" \
  -d '{
    "url":"https://example.com",
    "device":"phone",
    "width": 390,
    "height": 844,
    "format":"webp",
    "quality": 80,
    "zoom": 1.0,
    "element_selector": ".site-branding,#post-77,#post-76",
    "remove_selectors": ".cookie-banner,#subscribe-modal,header.sticky,img",
    "wait_for_selector": "body",
    "wait_for_state": "visible",
    "wait_for_timeout_ms": 5000,
    "delay_ms": 300
  }'
```

---

## Endpoint: `GET /result`

Fetch task status and (when ready) the screenshot.

### Request

* **Method:** `GET`
* **Query:** `id` (required)

Example:

```bash
curl -s "https://screenshot-api8.p.rapidapi.com/result?id=9f0d2b5b4c3a1e..." \
  -H "X-RapidAPI-Key: {RAPIDAPI_KEY}" \
  -H "X-RapidAPI-Host: screenshot-api8.p.rapidapi.com"
```

### Responses

#### Pending

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "pending"
}
```

#### Done

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "done",
  "content_type": "image/jpeg",
  "image_base64": "/9j/4AAQSkZJRgABAQ..."
}
```

#### Error

```json
{
  "task_id": "9f0d2b5b4c3a1e...",
  "status": "error",
  "error": "Failed to capture screenshot"
}
```

#### Missing `id`

```json
{
  "status": "error",
  "error": "Parameter 'id' is required"
}
```

---

## How to decode `image_base64`

The `done` response returns the image file as Base64.

* Decode Base64 → get raw bytes
* Save bytes to a file with the proper extension (`.jpg`, `.png`, `.webp`) based on `content_type`

Example (Linux):

```bash
# Suppose you saved the JSON response to result.json and extracted image_base64 into image.b64
base64 -d image.b64 &gt; screenshot.webp
```

---

## Status codes (inside JSON)

* `accepted` — task created successfully (returned by `/create`)
* `pending` — still processing (returned by `/result`)
* `done` — finished successfully (returned by `/result`)
* `error` — failed (returned by `/create` or `/result`)

---

## Limits & Rules (user-facing)

To keep the service fast and safe, some limits apply:

* **Allowed URL schemes:** `http`, `https` only
* **Blocked targets:** localhost / private network IPs (SSRF protection)
* **Max viewport width:** `3840`
* **Max viewport height:** `2160`

  * If you request a larger `height`, the API will treat it as a “tall/full-page” intent and will return a cropped full-page result within safe limits.
* **Max output height:** capped (very tall pages are cropped)
* **Max matched elements for `element_selector`:** limited (if your selector matches too many elements, you’ll get an error)
* **Max `delay_ms` / waits:** up to `30000` ms (values above are clamped)
* **Max output size:** very large images may be rejected

---

## Recommended polling strategy

After `POST /create`:

* Poll `GET /result?id=...` every **0.5–2 seconds**
* Stop polling when you receive `done` or `error`

---

## Final notes

Screenshot API is built to be simple: **create → poll → decode**.
Use `remove_selectors`, `wait_for_selector`, and `delay_ms` to tame real-world websites (cookie banners and “pop up now!” UI are a fact of life).

May your screenshots be crisp, your selectors be correct, and your popups be removed without mercy.
