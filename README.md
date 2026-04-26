# UTM Grabber

Lightweight JavaScript library that automatically captures UTM parameters and cookies from the current page, persists them across navigation via `sessionStorage`, and injects them into form fields and outgoing links.

**CDN (latest):**
```html
<script src="https://cdn.jsdelivr.net/gh/dkndn/utmgrabber@latest/utmg.min.js"></script>
```

---

## How it works

1. On every page load, UTM Grabber reads all URL parameters and first-party cookies and stores them in `sessionStorage`.
2. If the current page has no UTM params but the referring page (same root domain) did, those params are used as a fallback.
3. If neither source nor medium can be determined, defaults of `utm_source=direct` / `utm_medium=none` are set.
4. Once the DOM is ready, all configured form fields and URLs are enriched with the stored parameter values.
5. A `MutationObserver` watches for late-rendered DOM nodes (popups, JS-injected forms) and re-runs the injection automatically.

---

## Quick start

### Without configuration (zero-config)

Just include the script. UTM Grabber will run automatically with sensible defaults: it captures URL params and injects them into all external anchor links and any `<input name="...">` fields that match stored UTM keys.

```html
<script src="https://cdn.jsdelivr.net/gh/dkndn/utmgrabber@latest/utmg.min.js"></script>
```

### With configuration

Create a config file that listens for the `utmgrabber/ready` event:

```html
<script src="https://cdn.jsdelivr.net/gh/dkndn/utmgrabber@latest/utmg.min.js"></script>
<script>
    document.addEventListener('utmgrabber/ready', function() {
        window.UtmGrabber.setConfig({
            grab: {
                cookies: [
                    { name: 'fbclid' }
                ],
                urlparams: [
                    { name: 'utm_source' },
                    { name: 'utm_medium' },
                    { name: 'utm_campaign' },
                    { name: 'utm_term' },
                    { name: 'utm_content' }
                ]
            },
            storeIn: {
                urls: [
                    {
                        url: 'checkout.example.com',
                        positions: { href: true }
                    }
                ],
                forms: []
            }
        });
    });
</script>
```

---

## Configuration reference

### `setConfig( config )`

Sets the library configuration, triggers parameter collection, and starts DOM injection.

```js
UtmGrabber.setConfig({
    logging: true,           // optional: enable console logging

    grab: {
        cookies: [           // cookies to capture
            { name: 'my_cookie' }
        ],
        urlparams: [         // URL parameters to capture
            { name: 'utm_source' },
            { name: 'utm_medium' },
            { name: 'utm_campaign' }
        ]
    },

    storeIn: {
        urls: [ /* StoreUrlsConfig[] */ ],
        forms: [ /* StoreFormsConfig[] */ ]
    }
});
```

---

### `storeIn.urls` — URL configuration

Appends stored params to matching links on the page.

```js
storeIn: {
    urls: [
        {
            url: 'example.com',          // matches all links containing this string
            appendParams: [              // optional: only append these specific params
                'utm_source', 'utm_medium'
            ],
            positions: {
                href: true,              // <a href="..."> tags
                iFrame: true,            // <iframe src="..."> tags (applied on window.load)
                dataAttr: true           // elements with [data-url] attribute
            },
            spg: ['utm_data']            // optional: Single Param Grouping key (see below)
        }
    ]
}
```

**Auto-fallback:** Even without any `urls` configuration, UTM Grabber automatically appends UTM parameters to all *external* anchor links (`<a href="...">`).

---

### `storeIn.forms` — Form field configuration

By default, UTM Grabber matches `<input name="utm_source">` etc. by their `name` attribute. Custom form configs allow you to define your own field selection logic:

```js
storeIn: {
    forms: [
        {
            fieldsSelector: 'input.hidden-utm',
            fieldKey: (inputEl) => inputEl.getAttribute('data-utm-key') || ''
        }
    ]
}
```

**Value-token pattern:** If an input field already has a value that matches a UTM key (e.g. `value="utm_source"`), UTM Grabber replaces it with the actual stored value. This supports tools that use the `value` attribute as a placeholder token.

---

### Single Param Grouping (SPG)

Serializes all UTM params into a single URL parameter as a `key:value|key:value` string. Useful for third-party embeds that only accept one tracking parameter.

```js
{
    url: 'embed.example.com',
    spg: ['utm_data']   // writes all UTM values into ?utm_data=utm_source:google|utm_medium:cpc|...
}
```

The values are URL-encoded and special characters (`\`, `:`, `|`) are escaped.

To receive the values as JSON instead, use `UtmGrabber.getSingleParamGroupingVals('json')`.

---

## Events

| Event | Fires when |
|---|---|
| `utmgrabber/ready` | DOM is ready, but before `setConfig()` is called. Use this to call `setConfig()`. |
| `utmgrabber/configloaded` | `setConfig()` has been called and DOM injection has started. |

```js
document.addEventListener('utmgrabber/ready', function() {
    UtmGrabber.setConfig({ ... });
});

document.addEventListener('utmgrabber/configloaded', function() {
    console.log('UTM Grabber is active');
});
```

---

## API

### Configuration

| Method | Description |
|---|---|
| `setConfig( config )` | Sets the full library config and starts injection. |
| `addUrlConfig( urlConfig )` | Adds a URL config after initial setup and immediately processes matching links. |
| `addFormConfig( formConfig )` | Adds a form config after initial setup and immediately processes matching fields. |
| `setLogging( true\|false )` | Enables or disables console logging. |

### Storage

| Method | Description |
|---|---|
| `getStorageValue( key )` | Returns the stored value for a given key, or `''`. |
| `setStorageValue( key, value, overrideIfExists? )` | Writes a value to storage. Pass `false` as third argument to skip if key already exists. |
| `getStoredParamKeys()` | Returns all keys currently in storage. |
| `getStoredForwardableKeys( 'url'\|'all' )` | Returns stored keys filtered to URL-forwardable ones (`utm_*`, `fbclid`, `gclid`) or all. |

### Utilities

| Method | Description |
|---|---|
| `getAllImportantParamStrings()` | Returns all URL-forwardable keys currently in storage. |
| `getSingleParamGroupingVals( format?, params? )` | Serializes stored params as `key:value\|...` or JSON. |
| `appendParamsToUrl( url, params, spg )` | Appends stored params to a URL string. Returns the updated URL. |
| `isForwardableUrlKey( key )` | Returns `true` if the key is a UTM param, `fbclid`, or `gclid`. |
| `isExternalUrl( url )` | Returns `true` if the URL points to a different hostname. |

---

## Referrer fallback

If the landing page has no UTM parameters in the URL, UTM Grabber checks whether the HTTP referrer belongs to the **same root domain** (e.g. `sub.example.com` → `example.com`). If so, the referrer's query parameters are used as a fallback — without overwriting any already-stored values.

---

## Browser support

Works in all modern browsers. Requires `sessionStorage`, `URLSearchParams`, `URL`, and `MutationObserver` — all available in every major browser since 2017.

---

## Release / CDN versioning

```bash
npm run release -- 1.2.3
```

Builds `utmg.min.js`, commits it, tags the release, and pushes to GitHub. jsDelivr picks up the new `@latest` version automatically.

| URL | Use case |
|---|---|
| `@latest` | Always the newest released version |
| `@v1.2.3` | Pinned to a specific version |
