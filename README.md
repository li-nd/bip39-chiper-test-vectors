# Test vectors

Machine-readable conformance vectors for **Positional BIP-39 Obfuscation, format v1**.

Third-party implementations should pass all checks in these files to claim v1 compatibility with Bip39Chiper.

**Self-contained bundle:** copy this entire `test-vectors/` folder into another project. It includes the normative spec (`v1/algorithm-spec-v1.md`) and a verification guide (`CONFORMANCE.md`).

## Layout

```
test-vectors/
  README.md                 — this file
  CONFORMANCE.md            — how to verify an implementation (step-by-step)
  v1/
    manifest.json           — format version, alphabet, radix, file index
    algorithm-spec-v1.md    — normative algorithm specification (bundled copy)
    kdf.json                — PBKDF2-HMAC-SHA256 key derivation (80 cases)
    tokens.json             — single-token HMAC + encoding (100 cases)
    obfuscate.json          — full mnemonic → token list (96 cases)
    recovery.json           — shuffled token recovery (288 cases)
    normalize.json          — token normalization (70 cases)
    export.json             — text export file parsing (15 cases)
    export-files/           — sample `.txt` exports referenced by export.json
```

**Total: 649 vector cases** (634 crypto/normalize + 15 export).

## Verifying in another project

1. Read [`CONFORMANCE.md`](CONFORMANCE.md) — layer-by-layer verification steps and JSON field reference.
2. Read [`v1/algorithm-spec-v1.md`](v1/algorithm-spec-v1.md) — normative algorithm definition.
3. Run all cases in order: KDF → tokens → obfuscate → recovery → normalize → export.

## File formats

All JSON files are UTF-8 arrays of objects unless noted in `manifest.json`.

| File | Validates |
|------|-----------|
| `kdf.json` | `DeriveKey(password, salt, iterations, keyBytes)` — multiple passwords, iteration presets, key sizes 16/32/64 |
| `tokens.json` | Full token pipeline for positions 1–24 and word indices across the wordlist |
| `obfuscate.json` | End-to-end obfuscation for N ∈ {12,15,18,21,24}, 16 entropy seeds each, plus password/iteration variants |
| `recovery.json` | Lookup-table recovery with reverse, rotate-half, and odd-even shuffles |
| `normalize.json` | Token normalization, separators, invalid characters, length edges |
| `export.json` | Text export file parsing + token line (15 `.txt` samples in `export-files/`) |

## Alphabet note

Format v1 uses a **30-character** alphabet (radix 30), not 32. See [Algorithm specification](v1/algorithm-spec-v1.md) §5.
