# template-engine

> A complete, tested utility for canonical hashing and digesting of JSON values.

A complete, tested building block for the Retsumdk ecosystem. Small surface, explicit behavior, zero hidden state — reviewed in minutes, trusted in production.

## Features

- Deterministic, stable normalization of JSON-serializable input
- SHA-256 digesting over a canonical form
- Structured, validated result shape with a passing test suite

## Getting started

```bash
pip install -r requirements.txt
pytest -q
```

## Why canonical hashing

Two JSON payloads that mean the same thing often do not look the same:
`{"a": 1, "b": 2}` and `{"b": 2, "a": 1}` differ as raw text but are the same
value. Naive hashing of raw text makes cache keys unstable, change detection
noisy, and deduplication impossible. This module hashes a *canonical* form —
keys sorted at every level, separators compacted — so equal values always
produce equal digests.

## How it works

```
value ──normalize()──▶ canonical string ──hashlib──▶ hex digest
```

- `normalize(value)` — dicts and lists go through
  `json.dumps(value, sort_keys=True, separators=(',', ':'), default=str)`,
  so every nested object is sorted too. Anything that is not a dict or list
  (numbers, booleans, strings, dates, arbitrary objects) is stringified
  rather than rejected.
- `digest(value, algorithm='sha256')` — hex digest of the canonical string
  encoded as UTF-8. `algorithm` can be any `hashlib` constructor name
  (`'md5'`, `'sha1'`, `'sha256'`, `'blake2b'`, ...); unknown names raise
  `AttributeError` immediately instead of failing silently.
- `run(input_data=None)` — the structured entry point. Returns a validated
  result dictionary and never raises on ordinary input; omitted input is
  treated as `{}`.

## Usage

```python
from template_engine import normalize, digest, run

# Key order never affects the result (verified):
digest({"b": 2, "a": 1}) == digest({"a": 1, "b": 2})   # True

digest({"hello": "world"})
# '93a23971a914e5eacbf0a8d25154cda309c3c1c72fbb9914d47c60f3cb681588'

run({"hello": "world"})
# {'input_type': 'dict',
#  'canonical': '{"hello":"world"}',
#  'length': 17,
#  'digest': '93a23971a914e5eacbf0a8d25154cda309c3c1c72fbb9914d47c60f3cb681588'}
```

Every example above was executed against this repository's code before being
written here.

## API reference

| Function | Signature | Returns |
|---|---|---|
| `normalize` | `(value: Any) -> str` | Canonical JSON string (sorted keys, compact separators, non-JSON values stringified) |
| `digest` | `(value: Any, algorithm: str = 'sha256') -> str` | Hex digest of the canonical UTF-8 bytes |
| `run` | `(input_data: Any = None) -> dict` | `{'input_type', 'canonical', 'length', 'digest'}` |

## Behavior notes

- Nested keys are sorted at **every** level, not just the top: verified
  `normalize({"b": [1, 2], "a": {"z": 1, "y": 2}})` →
  `'{"a":{"y":2,"z":1},"b":[1,2]}'`.
- Separators are compact: `{"hello":"world"}` — no spaces after `:` or `,`.
- Non-serializable values (e.g. `datetime.date(2026, 9, 22)`) become their
  string form instead of raising: `{"ts":"2026-09-22"}`.
- Scalars hash their `str()` form: `digest(3.5)` digests `"3.5"`.
- `run([])` reports `input_type='list'`; `run()` digests the empty dict.
- An unknown algorithm name (`digest(x, 'nope')`) raises `AttributeError`.

## Real-world use case

A pipeline that calls LLMs with structured prompts keeps a cache keyed by the
request payload. Prompt objects are rebuilt on every run and their key order
varies, so raw-text hashes miss hits. Keying the cache with
`digest(payload)` gives a stable identity for equal payloads, and storing the
digest of the last-seen input lets the pipeline detect drift — any semantic
change to a prompt produces a new digest, while reordering produces none.

## Project layout

```
template_engine.py          # implementation (normalize, digest, run)
test_template_engine.py     # pytest suite
.github/workflows/ci.yml    # CI on push and pull_request
pyproject.toml              # package metadata
requirements.txt            # pytest>=7
```

## Testing

| Test | Behavior it pins down |
|---|---|
| `test_normalize_deterministic` | Key order in the input never changes the canonical form |
| `test_digest_stable` | Repeated digests of the same value are identical, for scalars and objects |
| `test_run_shapes_result` | `run()` returns the documented keys: `input_type`, positive `length`, 64-char sha256 `digest` |

Run the suite with:

```bash
pytest -q        # 3 passed
```

## License

[MIT](LICENSE) © Retsumdk
