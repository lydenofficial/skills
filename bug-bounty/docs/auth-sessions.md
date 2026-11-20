# Auth Sessions — Full Guide

Anonymous recon misses the bugs that pay most. IDOR, BOLA, mass-assignment,
privilege escalation, auth bypass, SSRF behind login, and most LLM/agent
bugs are invisible until you log in. Load auth **once** at session start
and every downstream tool (httpx, katana, ffuf, nuclei, dalfox, the SQLi /
SSTI / upload PoC verifiers) sends those headers automatically.

Implemented in `bughunter/tools/auth_session.py` (`AuthSession`).

## Three ways to load auth

```bash
# 1. CLI flags (one hunt)
python3 bughunter/tools/hunt.py --target T --cookie 'session=eyJabc...'
python3 bughunter/tools/hunt.py --target T --bearer 'eyJhbGciOi...'
python3 bughunter/tools/hunt.py --target T --auth-file .private/T.json
python3 bughunter/tools/hunt.py --target T --auth-header 'X-Api-Key: abc123'

# 2. Environment (persists for the shell)
export BBHUNT_COOKIE='session=eyJabc...'
export BBHUNT_BEARER='eyJhbGciOi...'
export BBHUNT_API_KEY='abc123'
export BBHUNT_AUTH_HEADER='X-Custom: value'   # repeatable via newlines
python3 bughunter/tools/hunt.py --target T

# 3. JSON auth file (preferred for repeat hunts)
python3 bughunter/tools/hunt.py --target T --auth-file .private/T.json
```

## Auth file format

`docs/auth.example.json` is the template. Two accepted shapes:

```json
{"headers": ["Cookie: session=...", "X-Foo: bar"]}
```

```json
{"cookie": "session=...", "bearer": "eyJ...", "api_key": "..."}
```

A `.env`-style `key=value` file is also accepted. Headers are validated
(`Name: value`, no CR/LF — header injection is refused, not sent) and
deduplicated by case-insensitive name (last write wins).

## IDOR / BOLA: hunt with two sessions

Load two different users and diff behavior — the same endpoint answering
differently per identity is the IDOR signal:

```bash
python3 bughunter/tools/hunt.py --target T --auth-file .private/T-user-a.json
python3 bughunter/tools/hunt.py --target T --auth-file .private/T-user-b.json
# Audit log entries carry different session_id hashes → diff which
# endpoints behaved differently per identity.
```

## Safety rules (non-negotiable)

- **Cookies/tokens never appear in logs, hunt-memory, or `repr()`.**
  Only a 12-char `session_id` hash is recorded:
  `session_id = sha256(sorted canonical headers)[:12]`.
- **Auth files live under `.private/`** and `.private/` is gitignored.
  Never commit a real auth file, never paste a real token into a report,
  a chat log, or a PoC.
- **MFA-skip and SAML signature-stripping probes deliberately stay
  anonymous** — that's the attack they're checking for. Do not attach
  a real session to a probe that tests whether auth can be skipped.
- Rotate or revoke the test accounts' sessions when the hunt is done.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Tool sends requests without auth | Auth was loaded after the tool started, or the flag/env name is misspelled. Load auth at session start; check `BBHUNT_SESSION_ID` is set in the shell. |
| `invalid header (expected 'Name: value')` | The header string is malformed. Fix the auth file / flag. |
| `header contains CR/LF — refusing` | The value contains a newline (often a copy-paste artifact). Strip it. |
| Same `session_id` for two users | The two auth files resolve to identical headers — one of them didn't load (wrong path) or both users share a token. Check the file paths. |
| 401 on every request | The session expired or the cookie is for the wrong domain. Re-login in the browser and re-export the cookie. |
