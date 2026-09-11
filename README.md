# Tetrapolar Key Recovery Tool

A standalone, offline, single-file HTML page that decrypts a Tetrapolar Simple Key
backup (`.tpkey`) and reveals your seed phrase. Everything runs locally in your
browser — open the file on an air-gapped machine, drop in your backup, type your
recovery password, get your seed.

**The deliverable is one file: `tetrapolar-key-recovery.html`. Double-click to use.**

## Why a single offline file

A seed phrase is the most sensitive secret a wallet user owns. This tool is built so
that using it requires trusting as little as possible:

- **Zero network.** No CDNs, fonts, analytics, or telemetry — all cryptography
  (Argon2id WASM, XChaCha20-Poly1305, the BIP39 wordlist) is inlined into the file.
  A strict Content-Security-Policy (`default-src 'none'`) makes any outbound
  request impossible even in the presence of a bug. DevTools' Network tab will
  show zero requests for the entire session.
- **No persistence.** The password, derived key, and seed are never written to
  storage, the URL, or the console.
- **Zeroization.** Key and plaintext byte arrays are overwritten after use; Clear /
  closing the page wipes state, DOM, and clipboard.

**Recommended usage:** download the file, disconnect from the internet (or use a
machine that never goes online), open it, recover your seed, then delete the file.

## Using it

1. Open `tetrapolar-key-recovery.html` in a desktop browser (Chrome, Firefox,
   Safari, Edge) — `file://` is fine.
2. Drop your `.tpkey` backup file (or click to browse). The tool shows the backup's
   metadata: creation date, fingerprint, cipher, KDF parameters.
3. Type your recovery password and click **Decrypt**. Key derivation takes
   a few seconds by design (Argon2id at 64–256 MiB).
4. Click **Reveal seed phrase** and write the words down. Use **Copy** if needed.
5. Click **Clear and start over** when done.

A wrong password and a corrupted backup produce the same error
("Incorrect password or corrupted backup") — this is deliberate; the tool never
reveals which one failed.

> Low-memory mobile browsers may fail at 256 MiB KDF settings. If that happens,
> use a desktop browser.

## How it works

A `.tpkey` file is a JSON envelope (`tetrapolar-encrypted-key`, version 1). The tool
accepts both the raw envelope and the wrapped server response
(`{ "success": true, ..., "envelope": { ... } }`).

Decryption pipeline:

1. **Validate** the envelope (format, version, KDF/cipher names, parameter bounds).
2. **Derive the key** — Argon2id over the NFC-normalized UTF-8 password, using the
   salt and parameters recorded in the envelope, 32-byte output used directly as
   the AEAD key.
3. **Decrypt** — `aes-256-gcm` (WebCrypto) or `xchacha20-poly1305`
   ([@noble/ciphers](https://github.com/paulmillr/noble-ciphers)), 16-byte tag
   appended to the ciphertext. The **AAD** is the UTF-8 encoding of a fixed-order
   JSON of the envelope metadata (format, version, backup_id, created_at,
   master_fingerprint, word_count, KDF params, cipher — excluding salt, nonce, and
   ciphertext), which binds the ciphertext to its metadata.
4. **Decode the payload** — the plaintext is a JSON payload
   (`payload_type: "bip39-entropy"`) containing base64 BIP39 entropy, which the
   tool converts to the seed phrase (SHA-256 checksum, English wordlist). If the
   plaintext isn't that payload, it's displayed as-is with a warning.
decrypted with the page's vendored stack, so a convention mismatch fails loudly.

## License

MIT — © Tetrapolar LLC. See [LICENSE](LICENSE).
