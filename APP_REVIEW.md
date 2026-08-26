# App Review notes template

Merge supports user-installed declarative feed connectors. A connector is a JSON document containing URL templates, an HTTPS domain allowlist, and HTML/JSON field mappings.

- Merge does not download or execute JavaScript, Wasm, native binaries, bytecode, or other executable plugin code.
- Connectors cannot invoke or extend native iOS APIs.
- All HTTP requests are performed by Merge and limited to HTTPS GET requests for domains disclosed before installation.
- v1 connectors cannot access cookies, authentication headers, Keychain, files, local network addresses, clipboard, contacts, photos, location, or other privacy-protected data.
- v2 connectors may declare a user-initiated OAuth-style login. Merge presents the system authentication UI, stores the returned credential only in Keychain, and generates a host-controlled Bearer header for the declared service. The connector never receives the credential or Keychain access.
- Users can sign out from the connector management screen; removing a connector removes its associated Keychain credential.
- Response size, request timeout, item count, URL scheme, host, and headers are validated by the host app.
- Connector output is normalized into the same local Article model used by built-in RSS subscriptions.

Relevant implementation: `Merge/PluginSystem.swift`.
