# Security policy - meshvpn enterprise

meshvpn connects machines and carries their traffic, so we treat vulnerabilities in it as urgent.

## Reporting a vulnerability

Please **don't open a public issue**. Report privately:

- by e-mail to **security@plugovr.ai**, or
- through GitHub: **Security → Report a vulnerability** on
  [PlugOvr-ai/meshvpn](https://github.com/PlugOvr-ai/meshvpn) (for the open-source core).

Please include the version (`meshvpn --version`), the platform, what an attacker needs (network
position, keys, an invite, ...) and what they gain, and if possible steps or a proof of concept to
reproduce it. You may write in English or German.

What happens next:

| Step | Within |
|---|---|
| We confirm we received the report | 3 working days |
| First assessment (severity, affected versions) | 10 working days |
| Fix released for critical/high severity | 30 days, usually much sooner |
| Fix released for medium/low severity | 90 days, or with the next release |

We keep you informed, agree the publication date with you, and credit you in the advisory unless
you prefer otherwise. We don't take legal action against research done in good faith: testing
only on your own systems or with permission, no access to other people's data beyond what is
needed to show the problem, no denial of service, and giving us reasonable time before disclosure.

## Supported versions

Security fixes are made for the **latest release**, for every customer with a valid license (and
during trials). Update with `meshvpn update` (or the installer); the portal is updated with
`install-portal.sh`. Versions older than the latest minor release don't get fixes. Customers are
told about critical fixes by e-mail.

## Advisories

Fixed vulnerabilities are named in the release notes of
[meshvpn-enterprise-releases](https://github.com/PlugOvr-ai/meshvpn-enterprise-releases), and
those in the open-source core as advisories of PlugOvr-ai/meshvpn (with a CVE where appropriate).

## What helps you check a release

- `SHA256SUMS` and a `.sha256` file for every archive of a release; `meshvpn update` checks them.
- A software bill of materials (CycloneDX, `meshvpn-enterprise-<version>.cdx.json`) with every crate the
  release is built from, to match against vulnerability databases.

## Scope

In scope: the enterprise `meshvpn` and `meshvpn-portal` programs, their installers, and the
license server licenses.plugovr.ai. Out of scope: vulnerabilities in the operating system,
in dependencies that are already fixed in a newer version we don't ship yet (tell us anyway), and
findings that need root access on the machine meshvpn runs on.
