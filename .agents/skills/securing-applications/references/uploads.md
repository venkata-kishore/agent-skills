# Uploads and downloads

## Upload

1. The client asks the server for an upload slot, stating file name, size and declared type.
2. The server checks access, the size limit and the allowed type list. It refuses otherwise.
3. The server returns a pre-signed URL with a short expiry, bound to one object key the server chose (never a client-supplied path).
4. The browser sends the file straight to object storage.
5. The client tells the server it is done. The server records the object only then.

- A pre-signed PUT cannot enforce the declared size or type. Bind size with signed conditions where the storage supports them (for example a POST policy with a content-length range), and on the completion call verify the stored object's size and type before recording it.
- The server never decodes or resizes untrusted images. Decoders are an attack surface (crafted files crash or exploit them).
- Treat the declared type as a hint. Do not serve user uploads from the application origin.

## Download

1. The server checks that the caller may read the object.
2. It returns a pre-signed link with a short expiry.
3. Serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`.
4. Never put the link in a log line or an audit row.
