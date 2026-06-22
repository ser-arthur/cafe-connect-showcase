# CafeConnect API

A stateless JSON REST API for the CafeConnect café directory. It exposes the same data as the web app and is meant to be called from code, a script or an API client.

## Overview

- Base URL: `https://cafe-connect.vercel.app`
- Every endpoint lives under the `/api` prefix.
- Requests and responses are JSON. Send `Content-Type: application/json` on any request that has a body.
- Access is authenticated with a signed JWT bearer token. Only `POST /api/login` is reachable without one.

Because every data endpoint requires an `Authorization` header and logging in is a `POST`, the API cannot be driven from a browser address bar. Use curl, an API client such as Postman, or code.

## Authentication

Authentication follows a two-step pattern.

1. Exchange credentials for a token. `POST /api/login` with an email and password returns a JWT.
2. Send the token on every other request as a header: `Authorization: Bearer <token>`.

`/api/login` authenticates an existing account only; it does not create one. Register through the web app first, then log in here for a token.

Tokens are signed with HS256 and expire one hour after they are issued. When a token expires, requests return `401`; log in again to obtain a fresh one. There is no separate refresh endpoint by design; the login call is the refresh.

## Access levels

| Capability | Endpoints | Who                            |
| --- | --- |--------------------------------|
| Obtain a token | `POST /api/login` | Any registered user |
| Read cafés | `GET /api/cafes`, `GET /api/cafes/{id}` | Any signed-in user             |
| Create a café | `POST /api/cafes` | Any signed-in user             |
| Update or delete a café | `PUT /api/cafes/{id}`, `DELETE /api/cafes/{id}` | Administrators only            |

## Rate limits

| Endpoint | Limit |
| --- | --- |
| `POST /api/login` | 10 per minute |
| `GET /api/cafes`, `GET /api/cafes/{id}` | 30 per minute |
| `POST /api/cafes` | 10 per minute |

Administrator tokens are exempt from these limits. Exceeding a limit returns `429`.

## Endpoints

### POST /api/login

Exchange credentials for a token. No authentication required.

Request `body`:
```json
{
  "email": "member@cafeconnect.app",
  "password": "Member@1234"
}
```

Response `200`:
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "message": "Login successful."
}
```

Errors: `400` (email or password missing), `401` (invalid email or password), `415` (Content-Type not JSON).

###
### GET /api/cafes

List cafés. Requires a token. With no query parameters, returns every café ordered by name. Every parameter below is optional, and they can be combined.

| Parameter | Description |
| --- | --- |
| `city` | Exact city match, case-insensitive, e.g. `city=London` |
| `country` | Exact country match in display form, e.g. `country=United Kingdom (UK)` |
| `min_wifi` | Minimum wifi strength, 0 to 5 |
| `sockets` | `true` to return only cafés with power sockets |
| `toilet` | `true` to return only cafés with a toilet |
| `food` | `true` to return only cafés serving food |
| `noise` | Noise level: Quiet, Moderate or Lively |
| `max_price` | Maximum coffee price, e.g. `max_price=3.50` |
| `min_rating` | Minimum average rating, 1 to 5, e.g. `min_rating=4` |
| `q` | Fuzzy text search over name, city and country |
| `near` | `lat,lng` to sort by distance, nearest first, e.g. `near=51.5074,-0.1278` |
| `radius_km` | Used with `near`, limits results to within this many kilometres |
| `sort` | `top` (rating), `price`, `name` or `newest`. Default `name`. Ignored when `near` is set |
| `limit` | Caps the number of cafés returned |
| `offset` | Skips this many cafés before returning |

Boolean parameters accept `true`, `1`, `yes` or `on`. Cafés without coordinates are omitted from `near` queries.

Response `200`:
```json
{
  "count": 1,
  "total": 1,
  "cafes": [
    { "id": 1, "name": "...", "city": "...", "...": "..." }
  ]
}
```

`count` is the number of cafés in this response; `total` is the number that matched the filters before any `limit` or `offset`. The two are equal unless you page. When `near` is used, each café also carries a `distance_km` field. Each item is a café object (see The café object). Errors: `401`.

Example queries:
```
GET /api/cafes?city=London&sockets=true&sort=top
GET /api/cafes?min_rating=4&max_price=3.50
GET /api/cafes?q=roastery
GET /api/cafes?near=51.5074,-0.1278&radius_km=5
GET /api/cafes?limit=20&offset=40
```

###
### GET /api/cafes/{id}

Fetch a single café by id. Requires a token.

Response `200`:
```json
{ "cafe": { "id": 1, "name": "...", "...": "..." } }
```

Errors: `401`, `404` (no café with that id).

###
### POST /api/cafes

Create a café. Requires a token, and is available to any signed-in user.

Request body: a café payload (see Café fields). Minimal example:
```json
{
  "name": "The Roastery",
  "map_url": "https://maps.google.com/?q=the-roastery",
  "city": "London",
  "country": "United Kingdom (UK)",
  "coffee_price": "3.20",
  "currency": "£",
  "seats": 40,
  "has_sockets": true,
  "has_toilet": true,
  "full_rating": 4
}
```

Response `201`:
```json
{ "message": "Cafe added successfully.", "cafe": { "id": 142, "...": "..." } }
```

Errors: `400` (validation failure, the body names the offending fields), `401`, `409` (a café with that name already exists), `415`.

Note: the API does not assign a placeholder image. If you omit `images`, the café is stored with none. To attach a photo, pass a public image URL in the `images` field. This differs from the web form, which fills in a representative image automatically for non-admin submissions.

###
### PUT /api/cafes/{id}

Update a café. Administrators only. Send only the fields you want to change.

Request body (partial):
```json
{ "coffee_price": "3.50", "noise_level": "Lively" }
```

Response `200`:
```json
{ "message": "The Roastery updated successfully.", "cafe": { "...": "..." } }
```

Errors: `400`, `401`, `403` (not an administrator), `404`.

###
### DELETE /api/cafes/{id}

Delete a café. Administrators only.

Response `200`:
```json
{ "message": "The Roastery deleted successfully." }
```

Errors: `401`, `403`, `404`.

###
## The café object

Every café returned by the API has the shape below. `images` is a comma-separated string of URLs, and may be empty. `average_rating` and `review_count` are computed from member reviews; when a café has no reviews, `average_rating` falls back to `full_rating`.

| Field | Type | Notes |
| --- | --- | --- |
| id | integer | |
| name | string | |
| map_url | string | Link to the café on a map |
| city | string | |
| country | string | Display form, e.g. "United Kingdom (UK)" |
| country_code | string | Two-letter code, derived from country |
| latitude | number or null | |
| longitude | number or null | |
| currency | string | Symbol, e.g. "£" |
| coffee_price | string | e.g. "3.20" |
| wifi_strength | integer | 0 to 5 |
| seats | integer | |
| has_sockets | boolean | |
| has_toilet | boolean | |
| has_food | boolean | |
| noise_level | string or null | Quiet, Moderate or Lively |
| opening_hours | string or null | |
| images | string | Comma-separated URLs, may be empty |
| full_review | string or null | Editorial summary |
| full_rating | integer | Baseline rating, 1 to 5 |
| created_at | string | ISO 8601 timestamp |
| average_rating | number | Mean of member reviews, or full_rating if none |
| review_count | integer | Number of member reviews |

Reviews themselves are created and read through the web app. The API exposes their aggregate (`average_rating` and `review_count`) on each café, but does not provide review endpoints.

## Café fields (create and update)

These are the fields accepted in a `POST` or `PUT` body. On create, the fields marked required must be present. On update, send any subset of them.

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| name | string | Yes | 2 to 120 characters, must be unique |
| map_url | string | Yes | Must be a valid URL |
| city | string | Yes | 2 to 80 characters |
| country | string | Yes | One of the supported countries below |
| coffee_price | string | Yes | A number written as a string, e.g. "3.50" |
| currency | string | Yes | One of the supported currency symbols below |
| seats | integer | Yes | 0 or greater |
| has_sockets | boolean | Yes | |
| has_toilet | boolean | Yes | |
| full_rating | integer | Yes | 1 to 5 |
| wifi_strength | integer | No | 0 to 5, default 0 |
| has_food | boolean | No | Default false |
| noise_level | string | No | Quiet, Moderate or Lively |
| opening_hours | string | No | Up to 120 characters |
| latitude | number | No | Between -90 and 90 |
| longitude | number | No | Between -180 and 180 |
| images | string | No | A public image URL, or several separated by commas |
| full_review | string | No | 3 to 300 characters |

Supported countries: Australia (AU), Canada (CA), Germany (DE), Spain (ES), France (FR), Ghana (GH), India (IN), Italy (IT), Japan (JP), Nigeria (NG), Netherlands (NL), Norway (NO), New Zealand (NZ), United Kingdom (UK), United States (US), South Africa (ZA).

Supported currency symbols: A$, C$, €, GH₵, ₹, ¥, ₦, kr, NZ$, £, $, R.

Supported noise levels: Quiet, Moderate, Lively.

## Status codes

| Code | Meaning here |
| --- | --- |
| 200 | Success |
| 201 | Café created |
| 400 | Malformed or empty body, or a validation failure |
| 401 | Missing, invalid or expired token, or a failed login |
| 403 | Signed in, but not an administrator |
| 404 | No café with that id |
| 409 | A café with that name already exists |
| 415 | Content-Type was not application/json |
| 429 | Rate limit exceeded |

## Testing

### curl

```bash
BASE="https://cafe-connect.vercel.app"

# 1. Log in as admin and capture the token (admin can reach every endpoint)
TOKEN=$(curl -s -X POST "$BASE/api/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@cafeconnect.app","password":"Admin@1234"}' \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['token'])")

# 2. List every café
curl -s "$BASE/api/cafes" -H "Authorization: Bearer $TOKEN"

# 3. List with filters: top-rated, quiet London cafés that have sockets
curl -s "$BASE/api/cafes?city=London&sockets=true&noise=Quiet&sort=top" \
  -H "Authorization: Bearer $TOKEN"

# 4. Fetch a single café by id
curl -s "$BASE/api/cafes/1" -H "Authorization: Bearer $TOKEN"

# 5. Create a café and capture the new id
NEW_ID=$(curl -s -X POST "$BASE/api/cafes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"The Roastery","map_url":"https://maps.google.com/?q=roastery","city":"London","country":"United Kingdom (UK)","coffee_price":"3.20","currency":"£","seats":40,"has_sockets":true,"has_toilet":true,"full_rating":4}' \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['cafe']['id'])")

# 6. Update that café (admin only): change the price and noise level
curl -s -X PUT "$BASE/api/cafes/$NEW_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"coffee_price":"3.50","noise_level":"Lively"}'

# 7. Delete it (admin only)
curl -s -X DELETE "$BASE/api/cafes/$NEW_ID" -H "Authorization: Bearer $TOKEN"
```

### Postman

1. Create a `POST` request to `{{base}}/api/login`. In the Body tab, choose raw and JSON, and paste the credentials. Send, then copy `token` from the response.
2. Create a `GET` request to `{{base}}/api/cafes`. In the Authorization tab, choose Bearer Token and paste the token. Send.
3. For `POST` and `PUT`, set the same Bearer token, then provide the JSON body in the Body tab.

Storing the base URL and token as Postman environment variables keeps the requests tidy.

### Python

```python
import requests

BASE = "https://cafe-connect.vercel.app"

# Authenticate
r = requests.post(f"{BASE}/api/login",
                  json={"email": "member@cafeconnect.app", "password": "Member@1234"})
token = r.json()["token"]
headers = {"Authorization": f"Bearer {token}"}

# Read
cafes = requests.get(f"{BASE}/api/cafes", headers=headers).json()["cafes"]
print(len(cafes), "cafes")
```

## Integration scenarios

A few common ways the API is consumed inside an application.

**Read-only data feed.** A dashboard, map or widget that displays cafés. Authenticate once on startup, call `GET /api/cafes` (optionally narrowing with filters such as `city` or `min_rating`), and render the result. Refresh the token roughly hourly, or whenever a call returns `401`.

**Member contribution.** A separate front end that lets people submit cafés. The client logs in with a member account, collects the café details through its own form, and posts them to `POST /api/cafes`. Validation errors come back as `400` with the offending fields named, which the client can surface against its own form.

**Administrative curation.** A back-office tool or maintenance script that corrects or removes entries. It logs in with an administrator account and uses `PUT` and `DELETE`. These are the only operations that require admin rights.

**Token lifecycle.** Because a token lasts one hour, a long-running client should treat a `401` as a signal to log in again and retry the request once. The example below wraps that pattern.

```python
import requests

BASE = "https://cafe-connect.vercel.app"
CREDS = {"email": "admin@cafeconnect.app", "password": "Admin@1234"}


class CafeConnect:
    def __init__(self, base, creds):
        self.base, self.creds = base, creds
        self.token = None

    def _login(self):
        r = requests.post(f"{self.base}/api/login", json=self.creds)
        r.raise_for_status()
        self.token = r.json()["token"]

    def _request(self, method, path, **kwargs):
        if self.token is None:
            self._login()
        headers = {**kwargs.pop("headers", {}), "Authorization": f"Bearer {self.token}"}
        r = requests.request(method, f"{self.base}{path}", headers=headers, **kwargs)
        if r.status_code == 401:            # token expired: refresh once and retry
            self._login()
            headers["Authorization"] = f"Bearer {self.token}"
            r = requests.request(method, f"{self.base}{path}", headers=headers, **kwargs)
        return r

    def list_cafes(self):
        return self._request("GET", "/api/cafes").json()["cafes"]

    def add_cafe(self, payload):
        return self._request("POST", "/api/cafes", json=payload).json()


client = CafeConnect(BASE, CREDS)
print(len(client.list_cafes()), "cafes")
```
