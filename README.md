# Google Calendar OAuth2 Integration (n8n)

Workflow: **Google Calendar OAuth2 Integration - Complete** (id `ycA7IM0pErSGVrBi`)

## 1. Purpose
Connects to Google Calendar using OAuth2 (authorization code flow). A caller posts event details, the user approves access on Google's consent screen, and the workflow creates the event on the user's primary calendar. If the access token has expired (HTTP 401), it refreshes the token and retries once.

## 2. Endpoints
| Purpose | Method | Production URL |
|---|---|---|
| Request auth link | POST | `https://mussabarshad383.app.n8n.cloud/webhook/cf02a62e-bdc8-418b-acae-c1141dd8d3a3` |
| OAuth callback (Google redirect URI) | GET | `https://mussabarshad383.app.n8n.cloud/webhook/8f3e9d2c-7b4a-4c5d-9e1f-2a8b3c4d5e6f` |

The callback URL is registered as the authorized redirect URI on the Google OAuth client `n8n-calendar-client`. Scope used: `https://www.googleapis.com/auth/calendar.events`.

## 3. Request body (POST)
```json
{
  "summary": "Team Meeting",
  "description": "Quarterly review",
  "start": "2025-12-20T10:00:00-07:00",
  "end": "2025-12-20T11:00:00-07:00",
  "attendees": ["boss@company.com"]
}
```

## 4. How it works

### Part A - Generate consent link (POST webhook)
1. **Webhook Trigger** receives the event details.
2. **Prepare State** (Code) encodes the body as JSON then base64. This `state` carries the event through Google's redirect.
3. **Build Auth URL** (Set) builds the Google consent URL: `client_id`, `redirect_uri`, `response_type=code`, scope, `access_type=offline` (so a refresh token is issued), `prompt=consent`, and the URL-encoded `state`.
4. **Respond to Webhook** returns `{ "authUrl": "..." }`.

The user opens `authUrl`, signs in and clicks Allow.

### Part B - Obtain tokens and create the event (GET callback)
Google redirects the browser to the callback URL with `code` and `state`.
1. **OAuth Callback** receives the redirect.
2. **Decode State** (Code) decodes `state` back into the event data and passes on the `code`.
3. **Exchange Code for Tokens** (HTTP POST to `https://oauth2.googleapis.com/token`, form-urlencoded, `grant_type=authorization_code`) returns an access token and a refresh token. A code can be used only once and expires quickly.
4. **Persist Refresh Token** (Code) saves the refresh and access tokens in the workflow static data.
5. **Create Calendar Event** (HTTP POST to the Calendar API `events` endpoint) sends `Authorization: Bearer <access_token>` and a JSON body with summary, description, start/end (time zone `America/Los_Angeles`) and attendees. Its error output feeds Part C.
6. **Success Response** then **Respond Callback Success** returns `{"status":"success","message":"Calendar event created successfully!"}` to the browser.

### Part C - Automatic token refresh
If Create Calendar Event fails:
1. **Token Expired?** (IF) checks for HTTP 401.
2. **True:** **Refresh Access Token** (HTTP POST, `grant_type=refresh_token`, using the saved refresh token) gets a new access token. **Retry Create Event** repeats the request. On success it goes to the success response.
3. **False**, or the retry fails: **Error Response** then **Respond Callback Error** returns the error details with HTTP 500.

## 5. Canvas groups
Generate consent link, Obtain tokens, Refresh and retry, Report failure, Confirm success.

## 6. Test results
| Execution | What ran | Result |
|---|---|---|
| 34, 36, 38 | Full flow with real Google consent | Success: tokens issued, event created |
| 35, 37 | Auth-link step | Valid `authUrl` returned |
| 30 | Callback with a fake code | `invalid_grant` at Exchange Code for Tokens (expected) |
| 19 | Early version | `invalid_grant: Malformed auth code`, caused by expressions missing `{{ }}`; fixed |

Not tested on real data: the 401 refresh branch (Part C). It is built and wired, but every real run had a valid token.

## 7. Notes and limitations
- Authorization codes are single-use; do not refresh or reopen the callback URL.
- `Exchange Code for Tokens` has no error route, so a bad or reused code ends the execution in an error.
- The Google `client_secret` is hardcoded in the Exchange Code for Tokens and Refresh Access Token nodes. Replace it with a placeholder before sharing the exported JSON.
- Tokens are held in workflow static data, which suits a demo but not production.
- Event time zone is `America/Los_Angeles`, taken from the example document.
- No alert email is sent on failure (optional in the assignment).
