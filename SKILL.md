---
name: init-py
description: Author an interactive `init.py` bring-up script for a Docker Compose stack — collects env vars via questionary, writes them to `.env` via python-dotenv (preserving comments), validates upstream connectivity with timeout-bounded probes, and supports idempotent `check` / `reconfigure KEY` subcommands. Use when the user asks for an "init.py", "bring-up tool", ".env wizard", "compose bootstrapper", or wants to scaffold a single-file Python CLI that prepares a host before `docker compose up`. Trigger words: "create init.py", "interactive .env setup", "bring-up script".
---

# init.py — interactive Docker Compose bring-up tool

A single, self-contained Python script that lives at a repo root and prepares the host before `docker compose up`:

1. Prompt the operator for each env var (URLs, secrets, paths).
2. Write them to `.env` (preserving comments, 0600 perms).
3. Run reachability probes against external services.
4. Support `check` (validate + probe, no writes) and `reconfigure KEY` (single field re-prompt) subcommands.

Distilled from a ~840-LOC reference implementation. Use the structure below verbatim — adapt the fields, not the shape.

## Top-of-file shape (verbatim)

```python
#!/usr/bin/env -S uv run --script
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "questionary>=2.0,<3",
#     "httpx>=0.27,<1",
#     "python-dotenv>=1.0,<2",
#     # add SDK probes you need (temporalio, redis-py, ...) — keep pinned ranges
# ]
# ///
"""<project> interactive bring-up tool.

Run on the production host before the first compose up:

  uv run init.py
  docker compose -f compose.prod.yml up -d --build

Idempotent. Re-runs surface every field; the current value in `.env` is shown
as the prompt default so pressing Enter keeps it. Secrets accept empty input
to mean "keep existing".

  uv run init.py check           # validate .env + run probes, no writes
  uv run init.py reconfigure KEY # force a fresh value for one field
"""
```

The `uv run --script` shebang + inline `# /// script` dep block is **the** entry pattern. It means the operator only needs `uv` installed on the host — no project venv, no `requirements.txt`, no pre-step.

## Required modules in the file

Roughly this carve-up (each separated by a banner comment):

1. **Constants** — `REPO_ROOT`, `ENV_FILE`, `COMPOSE_FILE`, `ENV_HEADER`.
2. **Pretty output** — `ok` / `warn` / `fail` / `info` / `section` helpers using ANSI codes, guarded by `sys.stdout.isatty()`.
3. **.env I/O** — `read_env(path)` and `update_env_file(path, updates)` via `dotenv.dotenv_values` / `dotenv.set_key`.
4. **Prompt wrappers** — a `_ask(q)` shim that raises `KeyboardInterrupt` when questionary returns `None`. Don't call `.ask()` directly anywhere else.
5. **Validators** — `non_empty` / `is_valid_url` / `is_valid_port` / `is_existing_file` — each returns `True` or an error string (the shape questionary expects).
6. **`collect_field(...)`** — the heart of the script. Handles existing-value-as-default, secret-keep-on-empty, placeholder detection, validator wiring. (Snippet below.)
7. **`step_<service>(env, updates)`** — one per service grouping. Each calls `section()` then 1+ `collect_field()` calls.
8. **Reachability probes** — `http_probe(url, headers=..., expected={...})` and any async SDK probes with `asyncio.wait_for` timeout watchdog. Composed by `run_probes(env)`.
9. **`_RECONFIGURE` registry** — `dict[KEY -> (label, validator, secret_bool)]`. Drives `reconfigure KEY`.
10. **`cmd_init` / `cmd_check` / `cmd_reconfigure` / `main`** — argparse glue, in that order.

## The `collect_field` skeleton

This is the only piece worth showing inline — most logic lives here. Copy it, then add fields by calling it from `step_*` functions.

```python
def collect_field(
    env: dict[str, str],
    updates: dict[str, str],
    *,
    key: str,
    label: str,
    secret: bool = False,
    validate: Optional[Callable[[str], bool | str]] = None,
    default: str = "",
    force: bool = False,
) -> str:
    """Prompt for KEY. Existing values surface as defaults; secrets allow Enter-to-keep."""
    current = env.get(key, "")
    has_current = bool(current) and not is_placeholder(current)

    if secret:
        if has_current and not force:
            # Existing secret → accept empty input as "keep". The validator
            # must allow empty in this mode; only run user-supplied validate
            # when there's actually input.
            def keep_or_validate(value: str) -> bool | str:
                if not value:
                    return True
                return validate(value) if validate else True

            value = prompt_password(
                f"{key} — {label} (Enter to keep existing)",
                validate=keep_or_validate,
            )
            if not value:
                return current
            updates[key] = value
            return value
        value = prompt_password(f"{key} — {label}", validate=validate or non_empty)
        updates[key] = value
        return value

    prompt_default = current if has_current else default
    value = prompt_text(f"{key} — {label}", default=prompt_default, validate=validate)
    if value != current:
        updates[key] = value
    return value
```

The two flags worth understanding:

- `is_placeholder(value)` returns `True` for empty strings AND for any `CHANGE_ME_*` marker. Lets a `.env.example` ship sentinels that get safely overwritten.
- `force=True` makes `reconfigure KEY` skip the "keep existing" path for secrets — the operator explicitly asked to overwrite.

## Anti-patterns — don't do these

### .env file handling

❌ **Don't roll your own .env writer.** Concatenating `f"{key}={value}\n"` and writing the file destroys comments, reorders keys, mishandles quoting. Use `dotenv.set_key`:

```python
set_key(str(path), key, value, quote_mode="auto", export=False)
```

❌ **Don't omit `quote_mode="auto"`** — the default in some dotenv versions quotes everything (`KEY="https://..."`) which is technically fine for compose but ugly. `auto` only quotes when whitespace / shell metachars are present.

❌ **Don't pass `export=True`** unless your `.env` is genuinely sourced by a shell. Compose `env_file:` reads `KEY=value` lines verbatim — `export KEY=value` works in bash but adds clutter.

❌ **Don't `chmod 0600` before writing.** The file might not exist yet. Pattern: create if missing → `set_key` for each update → `chmod` at the end.

❌ **Don't write the file when there are no updates.** Skipping the rewrite avoids touching mtime and prevents docker from thinking env changed.

### Prompt handling

❌ **Don't call `questionary.text(...).ask()` directly.** On Ctrl+C, `.ask()` returns `None` — your next line of code crashes trying to call `.strip()` on `None`. Wrap everything in `_ask(q)` that raises `KeyboardInterrupt`, and let `cmd_init` catch it once at the top.

```python
def _ask(q):
    result = q.ask()
    if result is None:
        raise KeyboardInterrupt
    return result
```

❌ **Don't expect `questionary.password()` to do "Enter = keep".** It will treat empty input as a value. You have to implement keep-on-empty yourself (see `collect_field` above). Otherwise re-runs force operators to retype every secret.

❌ **Don't use `questionary.password()` for non-secret IDs.** Public keys, queue names, etc. should be regular `text` prompts so the operator can see and edit them. Reserve password prompts for things that genuinely shouldn't echo (API keys, tokens, private keys).

### Reachability probes

❌ **Don't probe without a timeout watchdog.** A bad address (DNS hang, firewall blackhole, wrong port) will hang `init.py` forever. Wrap every SDK probe in `asyncio.wait_for(coro, timeout=15.0)`. For `httpx`, pass `timeout=5.0` to the request itself.

```python
def temporal_probe(env: dict[str, str]) -> tuple[bool, str]:
    try:
        return asyncio.run(asyncio.wait_for(_temporal_probe_async(env), timeout=15.0))
    except asyncio.TimeoutError:
        return False, "timeout after 15s"
    except KeyboardInterrupt:
        raise
    except Exception as exc:
        return False, f"{type(exc).__name__}: {exc}"
```

❌ **Don't treat 401/403/404 as failures.** They mean "reachable, auth/path wrong" — the operator's job, not init.py's. Accept them as "probe reached the server":

```python
ok_p, detail = http_probe(url, expected={200, 401, 403, 404})
```

❌ **Don't probe synchronously inline.** Always offer a "Run reachability probes now? (Y/n)" confirm; some environments can't reach external services at init time (air-gapped clusters, ports not yet opened).

### TLS / cert handling (if your stack uses mTLS)

❌ **Don't compare private-key bytes** to verify a cert/key pair. Different algorithms, different encodings. Compare `SubjectPublicKeyInfo` bytes derived from the cert public key and the private key's public key:

```python
cert_spki = cert.public_key().public_bytes(
    Encoding.PEM, PublicFormat.SubjectPublicKeyInfo
)
key_spki = priv.public_key().public_bytes(
    Encoding.PEM, PublicFormat.SubjectPublicKeyInfo
)
if cert_spki != key_spki:
    return "cert and key are not a matching pair"
```

❌ **Don't use a single `ca.verify(cert)` API.** RSA, EC, and Ed25519 each have different `verify()` signatures. Branch on `isinstance(ca_pub, rsa.RSAPublicKey | ec.EllipticCurvePublicKey | ed25519.Ed25519PublicKey)` and call the matching method.

❌ **Don't forget cert expiry / not-yet-valid.** Use `not_valid_after_utc` / `not_valid_before_utc` (cryptography ≥ 42) with the fallback to `.replace(tzinfo=timezone.utc)` for older certs.

### Subcommand wiring

❌ **Don't repeat field definitions across `step_*` and `_RECONFIGURE`.** They drift. The `_RECONFIGURE` registry is `{KEY: (label, validator, secret)}` — when you add a field to a `step_*`, add a registry row in the same diff. `reconfigure` reads from the registry, so missing keys silently fall back to "Unknown field".

❌ **Don't make `cmd_check` write.** Its job is "validate + probe, no side effects". Operators run it to debug stuck deploys. If it writes, they can't trust it.

### Compose interaction

❌ **Don't put secrets in compose.yml.** Keep them in `.env` (chmod 0600). Compose reads `.env` for `${VAR}` interpolation AND mounts via `env_file:` — both work; secrets stay in one place.

❌ **Don't rely on `host.docker.internal` resolving on Linux.** macOS gets it for free; Linux needs `extra_hosts: ["host.docker.internal:host-gateway"]` in the compose service. Document this when your bring-up tool produces compose configs.

❌ **Don't generate compose.yml from init.py.** Compose should be hand-committed. `init.py` only writes `.env`. Mixing the two means rollbacks need both files in sync.

## Suggested directory layout in the target project

```
<project>/
├── compose.prod.yml      # committed; reads .env
├── init.py               # this script (chmod +x)
├── .env.example          # CHANGE_ME_* placeholders, committed
├── .env                  # gitignored, written by init.py, mode 0600
└── certs/                # gitignored, populated by init.py if mTLS
```

`init.py` must be executable (`chmod +x init.py`) so `uv run init.py` works without `python`.

## Verification checklist for the generated script

Before declaring done, walk through these mentally on the script:

1. Re-running `uv run init.py` on a fully-populated `.env` should make zero changes (every field's current value is the prompt default; pressing Enter through all of them produces an empty `updates` dict).
2. `uv run init.py reconfigure SOME_KEY` should re-prompt only that key, then exit.
3. `uv run init.py check` should be read-only — verify by running with `chmod -w .env` first; it should not raise.
4. Ctrl+C at any prompt should print "aborted" and exit 130 cleanly; no traceback.
5. A bad probe target shouldn't hang the script for more than ~15s.
6. Secret prompts on a populated `.env` should accept Enter-to-keep without re-validating the current value.
7. Field added to `step_*` but missing from `_RECONFIGURE` — agent should flag this. Lint with: `grep "^\s*\"\w" init.py | grep RECONFIGURE` vs. set of keys used in `collect_field` calls.

## When the user has a specific service stack

Ask which services need:
- a URL field + http probe
- a Bearer token / API key (secret)
- mTLS cert material in `./certs/` (CA + client cert + client key)
- an SDK-level probe (e.g., `temporalio.Client.connect` + describe_namespace)

For each `(name, type)` pair the user names, emit one `step_<name>(env, updates)` function and one `_RECONFIGURE` entry per field. Reuse the validators (`is_valid_url`, `non_empty`, etc.) — don't write new ones for common shapes.
