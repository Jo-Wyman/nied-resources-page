# Claude Design prompt: connect a new page to the Wix CMS

Paste this into Claude Design when starting a new page for the site so it
wires up live CMS data the same way the Resources page does — via a Wix
HTTP Function, no API keys, no third-party backend.

---

```
This page must pull its content live from our Wix CMS using the same
pattern as our existing Resources page — no hardcoded/static content,
no third-party backend, and no API keys anywhere in the page.

Wire it up like this:

1. In the component's logic class (extends DCLogic), add a
   `RESOURCES_ENDPOINT`-style constant pointing to:
   https://www.nied.ca/_functions/{CMS_ENDPOINT_NAME}

   Replace {CMS_ENDPOINT_NAME} with the actual Wix HTTP Function name
   for this page's data (e.g. "eventsData", "teamData") — I'll tell you
   what it is, or default to a sensible name based on the content and
   flag it clearly as a placeholder if I haven't specified one.

2. In `componentDidMount()`, call an async `loadData()` method that
   fetches that endpoint, parses the JSON response, and stores the
   result(s) in state. Handle the failure case with a `loading` and
   `loadError` state, same as the existing page — show a loading
   message while fetching and a plain error message if the fetch fails
   (never a blank page).

3. In `renderVals()`, map the raw fetched records into the flat shape
   the template needs (resolve any referenced/included collections,
   compute display fields like colors or labels from raw CMS fields,
   don't put that logic in the template).

4. Do not include any API key, token, or secret anywhere in the page.
   This works because the Wix collection's Read permission is set to
   "Anyone" and the HTTP Function runs server-side on our Wix site —
   the fetch URL itself is not sensitive.

5. Assume the HTTP Function response will need a CORS header
   (Access-Control-Allow-Origin) allowing our GitHub Pages origin —
   don't add that in the frontend, just build the fetch as normal.
```

---

See [CMS-CONNECTION-GUIDE.md](CMS-CONNECTION-GUIDE.md) for the backend side
of this — creating the collection, setting its permissions, and writing the
matching Wix HTTP Function.
