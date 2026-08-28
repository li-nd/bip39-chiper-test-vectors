# Conformance verification guide

How to verify a **Positional BIP-39 Obfuscation v1** implementation against the bundled test vectors.

This folder is **self-contained**: copy `test-vectors/` (or just `test-vectors/v1/`) into any project together with `algorithm-spec-v1.md`.

## Quick start

1. Read `v1/algorithm-spec-v1.md` — normative algorithm definition.
2. Load `v1/manifest.json` — confirm `format_version`, `alphabet`, and `radix`.
3. Run checks in **layer order** (below). All must pass to claim v1 compatibility.

**Reference test runner (Swift):** `Bip39ChiperTests/TestVectorsTests.swift` in the Bip39Chiper repository.

---

## Prerequisites

Your implementation **must** provide:

| Dependency | Requirement |
|------------|-------------|
| BIP-39 wordlist | Official [English wordlist](https://github.com/bitcoin/bips/blob/master/bip-0039/english.txt) — 2048 words, line *k* → index *k − 1* |
| PBKDF2 | HMAC-SHA256 PRF (RFC 8018) |
| HMAC | HMAC-SHA256 (RFC 2104) |
| Integer encoding | Big-endian unsigned from truncated MAC bytes; radix-30 fixed-width encoding per spec §5.2 |

Password rules in vector tests: passwords are non-empty UTF-8 strings ≥ 8 characters (product constraint used by the reference app; KDF vectors use valid passwords).

---

## Verification order

Run layers **sequentially**. Later layers assume earlier ones are correct.

```
manifest → kdf → tokens → obfuscate → recovery → normalize → export
```

| Step | File | Cases | Assert |
|------|------|-------|--------|
| 0 | `manifest.json` | — | `format_version == "v1"`, `radix == 30`, alphabet matches implementation |
| 1 | `kdf.json` | 80 | `DeriveKey(password, salt, iterations, key_bytes)` → `key_hex` (lowercase hex, no separators) |
| 2 | `tokens.json` | 100 | Full pipeline: key, HMAC, truncation, encoding → `token`; optional intermediate checks |
| 3 | `obfuscate.json` | 96 | `Obfuscate(mnemonic, password, params)` → `tokens[]` |
| 4 | `recovery.json` | 288 | `Recover(tokens_shuffled, password, params, N)` → `expected_mnemonic` |
| 5 | `normalize.json` | 70 | `NormalizeToken(input)` → `normalized`; well-formed check → `valid` |
| 6 | `export.json` | 15 | Parse `.txt` file → headers + tokens match `obfuscate_id` source |

**Total: 649 vector cases** (634 crypto/normalize + 15 export).

---

## Shared JSON fields

All vector files are UTF-8 JSON **arrays of objects**.

### `params` block

Present in `kdf.json`, `tokens.json`, `obfuscate.json`, `recovery.json`:

```json
{
  "version": "v1",
  "salt_utf8": "Bip39Chiper.v1.positional-hasher",
  "iterations": 100000,
  "key_bytes": 32
}
```

Map to your parameter struct:

- `salt` = UTF-8 bytes of `salt_utf8`
- `iterations` = PBKDF2 iteration count (integer)
- `key_bytes` = derived key length (16, 32, or 64)
- `token_length` = 9 (fixed for v1, not in JSON)

---

## Layer 1 — KDF (`kdf.json`)

For each object:

```
key = PBKDF2-HMAC-SHA256(
  password = password_utf8 (UTF-8),
  salt     = params.salt_utf8 (UTF-8),
  iterations = params.iterations,
  dkLen    = params.key_bytes
)
ASSERT hex(key) == key_hex
```

**Example** (`kdf-p01-i01`):

| Field | Value |
|-------|-------|
| `password_utf8` | `testpassword` |
| `iterations` | `100000` |
| `key_bytes` | `32` |
| `key_hex` | `4539133836a67287348f10d8110cae6e9fa46b183c3b270ae73487042a41690b` |

---

## Layer 2 — Tokens (`tokens.json`)

For each object, verify in order:

### 2a. Key derivation

Same as Layer 1 using `password_utf8` and `params`. Assert `key_hex`.

### 2b. HMAC payload

```
payload_utf8 = "{version}:{position}:{word_index}"
             = e.g. "v1:1:0"
mac = HMAC-SHA256(key, payload_utf8)   // 32 bytes
ASSERT hex(mac) == hmac_sha256_hex
ASSERT hex(mac[0..5]) == hmac_truncated_hex
```

### 2c. Token encoding

```
token = EncodeFixed(mac[0..5], length=9)
ASSERT token == vector.token
ASSERT len(token) == 9
ASSERT every char in manifest.alphabet
```

Optional cross-checks (if your encoder exposes them):

- `truncated_big_endian_int` — big-endian integer from first 5 MAC bytes
- `value_mod_radix_pow_len` — that integer mod 30⁹
- `radix` = 30, `modulus` = `"19683000000000"`

### 2d. Wordlist (optional)

If `word` is present: `wordlist[word_index] == word`.

**Example** (`token-001-100k`):

| Field | Value |
|-------|-------|
| `position` | `1` |
| `word_index` | `0` |
| `word` | `abandon` |
| `payload_utf8` | `v1:1:0` |
| `token` | `3JKDEPFPN` |

---

## Layer 3 — Obfuscate (`obfuscate.json`)

For each object:

```
tokens = Obfuscate(mnemonic, password_utf8, params)
ASSERT tokens == vector.tokens
ASSERT len(tokens) == word_count == len(mnemonic)
```

Also verify `key_hex` matches derived key from password and params.

Mnemonic words are lowercase BIP-39 English. Length `N` ∈ {12, 15, 18, 21, 24}.

**Canonical IDs** (used by export samples):

- `obf-12-first-100k` — 12-word mnemonic, 100k iterations
- `obf-24-first-100k` — 24-word mnemonic, 100k iterations

---

## Layer 4 — Recovery (`recovery.json`)

For each object:

```
result = Recover(
  tokens     = tokens_shuffled,   // order intentionally permuted
  password   = password_utf8,
  params     = params,
  phraseLength = word_count
)
ASSERT result complete (all N slots filled)
ASSERT result mnemonic == expected_mnemonic
```

### Recovery algorithm (normative summary)

1. `key ← DeriveKey(password, params)`
2. `table ← BuildLookupTable(key, N, params)` — map each token to `(position, word, word_index)` for all `p ∈ 1..N`, `i ∈ 0..2047`
3. Parse input: split on whitespace / `,` / `;` / newlines; uppercase; **then** `NormalizeToken` (keep only alphabet chars)
4. For each token, look up in `table`; resolve ambiguous matches per spec §8.3
5. When all slots filled, validate BIP-39 checksum

Shuffle patterns in vectors:

| Suffix | Permutation |
|--------|-------------|
| `-reverse` | Last token first |
| `-rotate-half` | Second half, then first half |
| `-odd-even` | Odd indices, then even indices |

**Performance tip:** cache `table` across recovery vectors that share the same `(password, params, word_count)`.

---

## Layer 5 — Normalize (`normalize.json`)

For each object:

```
normalized = NormalizeToken(input)
ASSERT normalized == vector.normalized
ASSERT IsWellFormed(normalized) == vector.valid
```

Where `IsWellFormed(t)` means: after normalization, `len(t) == 9` and every character is in the alphabet.

Normalization rules (spec §5.3):

1. Uppercase input
2. Keep only characters from the 30-character alphabet
3. Do **not** pad or truncate to 8 — length may differ after filtering

Use default v1 params (`iterations=100000`, `key_bytes=32`) for the well-formed check unless your API requires explicit config.

---

## Layer 6 — Export files (`export.json` + `export-files/`)

For each object in `export.json`:

```json
{
  "id": "export-12-comma",
  "obfuscate_id": "obf-12-first-100k",
  "file": "export-files/sample-12-100k-comma.txt",
  "expected_version": "v1",
  "expected_word_count": 12,
  "expected_iterations": 100000,
  "expected_key_bytes": 32,
  "notes": "Comma-separated token line"
}
```

### Parse and assert

```
text = read_utf8(vector.file)
parsed = ParseExportFile(text)

if vector.expected_version is not null:
  ASSERT parsed.version == expected_version
else:
  ASSERT parsed.version is null

(same pattern for expected_word_count, expected_iterations, expected_key_bytes)

source = obfuscate[vector.obfuscate_id]
ASSERT ParseTokens(parsed.token_line) == source.tokens
```

### Covered variants (15 cases)

| Category | Examples |
|----------|----------|
| Word counts | N = 12, 15, 18, 21, 24 |
| Iterations | 100 000 and 600 000 |
| Passwords | `testpassword`, `securepassphrase` (via linked obfuscate ids) |
| Token separators | spaces, commas, semicolons, mixed |
| Header parsing | mixed-case keys (`VERSION`, `Words`, …) |
| Layout | blank lines before token row; tokens-only (no headers) |

### File format rules

```text
# version: v1
# words: 12
# iterations: 100000
# keyBytes: 32
TOKEN1 TOKEN2 ... TOKENN
```

- Comment lines start with `#`
- Header format: `key: value` (keys case-insensitive: `version`, `words`, `iterations`, `keybytes`)
- Token line: whitespace-, comma-, or semicolon-separated tokens
- Password must **not** appear

---

## Worked example (manual smoke test)

Use these values to debug a fresh implementation before running all 634 cases:

```
Password:     testpassword
Salt:         Bip39Chiper.v1.positional-hasher
Iterations:   100000
Key bytes:    32
Key (hex):    4539133836a67287348f10d8110cae6e9fa46b183c3b270ae73487042a41690b

Payload:      v1:3:42
Position 3, word index 42 → word "accident"
Token:        DRF92TRD
```

Full 12-word vector: see `obf-12-first-100k` in `obfuscate.json`.

---

## Claiming v1 compatibility

An implementation **may claim v1 compatibility** when:

- [ ] All 80 KDF vectors match exactly
- [ ] All 100 token vectors match exactly (including HMAC intermediates if checked)
- [ ] All 96 obfuscate vectors match exactly
- [ ] All 288 recovery vectors recover the expected mnemonic
- [ ] All 70 normalize vectors match
- [ ] All 15 export vectors parse and match their `obfuscate_id` token lists

Partial passes (e.g. only KDF + tokens) indicate progress but **not** full interoperability.

---

## Related files

| File | Role |
|------|------|
| `v1/algorithm-spec-v1.md` | Normative algorithm (bundled copy) |
| `v1/manifest.json` | Format metadata and file index |
| `README.md` | Package overview |
