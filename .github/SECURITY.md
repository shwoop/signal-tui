# Security Policy

## Supported versions

Only the latest release receives security fixes. Check the
[releases page](https://github.com/johnsideserf/siggy/releases) for the
current version.

| Version | Supported |
|---------|-----------|
| latest  | Yes       |
| older   | No        |

## Reporting a vulnerability

If you find a security issue in siggy, please report it privately:

1. **Do not open a public issue.** Security bugs need to be handled carefully
   to avoid exposing users before a fix is available.
2. **Use GitHub's private vulnerability reporting:**
   [Report a vulnerability](https://github.com/johnsideserf/siggy/security/advisories/new)
3. Alternatively, email the maintainer directly (see the GitHub profile for
   contact info).

Please include:
- A description of the issue and its potential impact
- Steps to reproduce or a proof of concept
- The version of siggy affected

## What to expect

- I will acknowledge your report within 48 hours.
- I will provide an initial assessment within 1 week.
- Fixes will be released as a patch version (e.g. v1.5.1) with credit to the
  reporter unless you prefer to remain anonymous.

## Scope

siggy is a TUI layer over [signal-cli](https://github.com/AsamK/signal-cli).
It does not implement cryptographic protocols or contact Signal servers directly.
Security issues in the Signal Protocol itself should be reported to the
[Signal team](https://signal.org/docs/).

Issues in scope for siggy include:
- Command injection or escape sequence injection
- Path traversal in attachment handling
- Information leakage (credentials, message content in logs/temp files)
- Denial of service via crafted input
- Any bypass of siggy's security features (incognito mode, debug redaction, etc.)
