# Connecting a static page to a Wix CMS collection (no outside services)

This guide sets up the exact pattern `index.html` already uses — a static,
publicly-hosted page fetching data from a **Wix HTTP Function**, backed
entirely by Wix's own Content Manager and hosting. No third-party proxy,
serverless platform, or API key involved.

```
[ index.html on GitHub Pages ]  --fetch()-->  [ https://{site}.com/_functions/{name} ]  --wixData.query()-->  [ Wix Content Collection ]
```

## Why no API key is needed

Wix HTTP Functions are just public endpoints hosted on your Wix domain. The
code behind them runs in your site's backend and can query any collection
with `wixData`. Security is enforced by **collection permissions** in the
Content Manager, not a key:

- If a collection's **Read** permission is set to **"Anyone"**, an
  unauthenticated `wixData.query()` call succeeds — including one triggered
  by a public visitor hitting your function.
- If Read is restricted (e.g. "Site Member" or "Admin"), the same query
  returns nothing/errors unless the code explicitly elevates permissions
  with `wixAuth.elevate()` — which you generally should NOT do for a
  function meant to be hit anonymously from an external static page.

**Rule of thumb:** only collections meant to be fully public should have
Read set to "Anyone." Keep Create/Update/Delete restricted (Admin-only) so
visitors can read but never modify data. Anything sensitive should live in
a separate, permission-restricted collection, not mixed into a public one.

## Step 1 — Create or identify the collection in Wix

In your Wix site's Content Manager:

1. Create the collection: `{NEW_COLLECTION_NAME}` (e.g. `Events`,
   `TeamMembers`, `FAQs`).
2. Add your fields, e.g. `{FIELD_1}`, `{FIELD_2}`, `{FIELD_3}`, and any
   reference fields pointing to other collections (like `Resources`
   references `Topics`, `Audiences`, `Provinces`, `ResourceTypes`).
3. Set the collection's read permission to **Anyone** so the HTTP Function
   can read it without elevation:
   - Open the collection in Content Manager.
   - Click the **Settings** (gear) icon in the collection's toolbar, or the
     **⋯** menu next to the collection name, and choose **Set permissions**.
   - Under **Content Permissions**, set:
     - **Who can read content:** Anyone
     - **Who can create/edit/delete content:** Admin (or whatever restricts
       editing to your team — leave this OFF "Anyone")
   - Click **Save**.

   This is the step that makes the whole pattern work without an API key —
   an anonymous `wixData.query()` call from the HTTP Function will fail
   with a permission error until Read is set to Anyone.

## Step 2 — Add a new HTTP Function

Open your site's backend code (`Backend` panel in the Wix Editor) and find
or create `http-functions.js` — this is the same file `resourcesData`
already lives in.

Add a new exported function. Wix requires the naming convention
`get_<name>` for a GET endpoint:

```js
import wixData from 'wix-data';
import { ok, serverError } from 'wix-http-functions';

export async function get_{ENDPOINT_NAME}(request) {
  try {
    const results = await wixData.query('{NEW_COLLECTION_NAME}')
      // .include('{REFERENCE_FIELD}') // if this collection references others
      .find();

    return ok({
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '{YOUR_GITHUB_PAGES_ORIGIN}' // e.g. https://{user}.github.io
      },
      body: {
        {RESPONSE_KEY}: results.items
      }
    });
  } catch (err) {
    return serverError({
      headers: { 'Content-Type': 'application/json' },
      body: { error: String(err) }
    });
  }
}
```

This becomes publicly reachable at:

```
https://{your-wix-domain}/_functions/{ENDPOINT_NAME}
```

(Wix strips the `get_` prefix from the URL path automatically.)

If this new data needs to combine with existing resource-page data in one
response, you can instead add it to the *existing* `get_resourcesData`
function — just add another `wixData.query('{NEW_COLLECTION_NAME}').find()`
to the `Promise.all([...])` already there, and include it as a new key in
the returned JSON body.

## Step 3 — CORS

Since `index.html` is hosted on GitHub Pages (a different origin than your
Wix domain), the response **must** include `Access-Control-Allow-Origin`
set to your GitHub Pages URL, as shown above — otherwise the browser will
block the fetch even though the request itself succeeds. Wix does not add
this header for you automatically on custom HTTP functions.

## Step 4 — Fetch it from the page

In the page's script (same place `RESOURCES_ENDPOINT` is defined), add:

```js
const {ENDPOINT_NAME_UPPER}_ENDPOINT = 'https://{your-wix-domain}/_functions/{ENDPOINT_NAME}';
```

And in `loadData()`, fetch it alongside (or instead of) the existing call:

```js
const res = await fetch({ENDPOINT_NAME_UPPER}_ENDPOINT);
if (!res.ok) throw new Error('bad response: ' + res.status);
const data = await res.json();
this.setState({ {STATE_KEY}: data.{RESPONSE_KEY} || [] });
```

Then map/use `{STATE_KEY}` wherever you need it in `renderVals()`, the same
way `topicsData`, `resourceTypesData`, etc. are mapped into `TOPICS`,
`TYPES`, and so on today.

## Step 5 — Test

1. Visit `https://{your-wix-domain}/_functions/{ENDPOINT_NAME}` directly in
   a browser — confirm it returns JSON with real data and no permission
   errors.
2. Open `index.html` (locally or via GitHub Pages) and check the browser
   console for CORS or fetch errors.
3. Edit a record in the Wix Content Manager and refresh — confirm the
   change shows up (there's no caching layer here, so it should be
   immediate).

## Step 6 — Publish

1. Publish your Wix site so the new HTTP Function goes live at its public
   URL.
2. Commit the updated endpoint URL / fetch code in `index.html` and push to
   GitHub — this is safe since there's no secret in it, only a public URL.
