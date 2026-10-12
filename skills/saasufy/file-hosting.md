# File Hosting

Saasufy provides basic HTTP/HTTPS file hosting functionality with support for client-side caching (with `ETag`).
This is especially useful for hosting images.

To upload files to Saasufy, you first need to create a field of type `string` on a model of your choice.
You will then need to check/enable the `blob` constraint to tell Saasufy to treat this field as a base64 file.

After you've deployed your schema from the Saasufy dashboard, you will be able to manually add new records into your model via its `data` section.
Saasufy will show you a file picker next to the relevant field name which you can use to upload a file/image to Saasufy.

After adding a record to Saasufy which holds a file in one of its fields, that file can be accessed over HTTP/HTTPS via your Saasufy service's `/files` HTTP endpoint.
The format of the URL to link directly to a specific file/image is:

```
https://:serviceId.saasufy.com/files/:modelName/:modelId/:fieldName
```
Each service serves its files on a subdomain of its own, named after the service.
The `:serviceId` is the `sid` part of the URL for your Saasufy service (which you get after deploying your service); if that URL is `wss://saasufy.com/sid7999/socketcluster/`, then your service ID is `sid7999`.

So if your file is stored inside an `Image` model with ID `58f2051c-14bc-4518-8816-ff387cfdd57e` inside a `data` field, you will be able to link to your image directly using this URL:

```
https://sid7999.saasufy.com/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data
```

You can use it to embed images into your application using the `<img>` tag like this:

```html
<img src="https://sid7999.saasufy.com/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data" alt="My image" />
```

Saasufy enforces access controls for HTTP in the same way as it does for its WebSocket protocol.
It's possible to block or restrict access to specific files stored on specific fields via the `Access` page under the relevant model.

If your model field's access is set to `restrict`, only authenticated users with matching permissions will be able to view/download the image/file.
In this case, the request must carry the user's JWT; see [Accessing Restricted Files](#accessing-restricted-files) below.

## Accessing Restricted Files

When a field's `accessRead` is set to `restrict`, send the user's signed JWT in an `Authorization: Bearer <signedAuthToken>` request header. That is how a request to the `/files` endpoint is authenticated. Without the header, or with a token which is invalid or expired, the request counts as unauthenticated and the endpoint answers `403`.

Note that this is the user's JWT obtained from the socket — not your admin `SAASUFY_API_KEY`, which is only for the Admin HTTP API under `https://saasufy.com/api/`.

You cannot attach headers to `<img src="...">`, so instead you fetch the file yourself and hand the element an object URL. The signed token is held by the `socket-provider`'s underlying socket:

```js
let socketProvider = document.querySelector('socket-provider');
let socket = socketProvider.saasufySocket;

// The token only exists once the socket has authenticated.
if (socket.authState !== 'authenticated') {
  for await (let event of socket.listener('authStateChange')) {
    if (event.newState === 'authenticated') break;
  }
}

async function loadRestrictedFile(url) {
  let response = await fetch(url, {
    headers: { Authorization: `Bearer ${socket.signedAuthToken}` }
  });
  if (!response.ok) {
    throw new Error(`Failed to load file: ${response.status}`);
  }
  return URL.createObjectURL(await response.blob());
}

let img = document.querySelector('img');
img.src = await loadRestrictedFile(
  'https://sid7999.saasufy.com/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data'
);
```

The resulting object URL can be assigned to `<img src>`, `<video src>`, `<a href download>` or a CSS `url(...)`.

Things worth knowing:

- **Revoke the object URL** when you're done — `URL.revokeObjectURL(img.src)` when tearing down the element or before replacing the source; otherwise each load leaks the file until the page unloads.
- **Client-side caching still works.** Saasufy sets `ETag` and `Cache-Control` on the response and `fetch` uses the normal HTTP cache, so a revisit revalidates and receives a `304` instead of re-downloading. The object URL itself only lives for the current page load, so keep a `Map` from file URL to object URL if you render the same file repeatedly.
- **Your origin has to be allowed.** A file is served on its own subdomain, so the fetch is cross-origin from every app — including one served on `saasufy.com` — and the browser preflights it because of the `Authorization` header. The endpoint answers that preflight according to the **Files allow origin** setting on your account's settings page: a comma-separated list of origins, `*` for any (the default), or empty to allow none. Changes take effect on the next deployment. Files which are not restricted need none of this: `<img src="...">` is not subject to CORS.
- **Handle `403` explicitly.** It covers 'not authenticated', 'not permitted' and 'token expired' alike, so it's the right place to trigger re-authentication rather than showing a broken image.
- The object URL is local to the page; it is not a shareable link. For genuinely shareable URLs, the field must not be restricted.

The same header works from `curl` or any server-side HTTP client, where CORS does not apply:

```bash
curl -H "Authorization: Bearer $SIGNED_AUTH_TOKEN" \
  'https://sid7999.saasufy.com/files/Image/58f2051c-14bc-4518-8816-ff387cfdd57e/data'
```
