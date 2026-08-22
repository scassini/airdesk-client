# Code Signing Policy

Free code signing provided by SignPath.io, certificate by SignPath Foundation.

## Project

airdesk is a rebranded distribution of [RustDesk](https://github.com/rustdesk/rustdesk),
an open source remote desktop client, licensed under AGPL-3.0. Source code, build
instructions, and this policy are published in this repository.

## Roles

| Role | Member(s) |
|------|-----------|
| Author (trusted committer) | Sylvain Cassini ([@scassini](https://github.com/scassini)) |
| Reviewer (external contributions) | Sylvain Cassini ([@scassini](https://github.com/scassini)) |
| Approver (signing decisions) | Sylvain Cassini ([@scassini](https://github.com/scassini)) |

airdesk is currently maintained by a single person. All three roles are held by
the same individual; this will be revisited and split across distinct people if
and when additional maintainers join the project.

## Privacy

airdesk does not collect telemetry or usage analytics beyond what is required to
establish a remote desktop connection (peer ID exchange with the configured
rendezvous/relay server, `rd.lejumo.com`). No personal data is collected, sold,
or shared with third parties. Users can self-host their own rendezvous/relay
server as documented in the upstream [RustDesk server guide](https://rustdesk.com/server).

## Signing Scope

Only binaries built from this repository's own source code are signed. Bundled
third-party/upstream libraries are not signed independently.
