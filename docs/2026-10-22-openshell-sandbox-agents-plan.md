---
date: 2026-10-22
author: Onur Solmaz <osolmaz@users.noreply.github.com>
title: Serve OpenShell sandbox agents
tags: [openshell, setup, telegram, github, tls]
---

# Serve OpenShell sandbox agents

## Goal

An agent that runs in an OpenShell sandbox uses stock `gh` and `git` against
GitHub through gh-broker. The sandbox holds no GitHub credential and, in the
default setup, not even its broker client secret. Approvals arrive in Telegram
and the buttons work for every broker. One `unyolo setup` run installs and
connects all of it, with no manual host steps.

## Why the current setup does not fit

The setup was designed for agents that run as isolated local Linux accounts:

- An agent proves its identity with a local Unix socket, guarded by a group,
  plus a client secret in its home directory.
- `sudo-broker` runs approved privileged commands as that agent's Linux user.
- Guided setup offers `local_account`, `container`, and `remote` agents, but it
  produces client files for an account or container, not a sandbox.

An OpenShell sandbox has no host account and cannot reach host Unix sockets.
Its network traffic leaves through the OpenShell proxy, which allows only
listed hosts, ports, and binaries. The proxy terminates TLS, injects provider
credentials into requests by replacing placeholder values, and opens a new TLS
connection upstream. It trusts only public roots for that upstream connection,
plus a CA bundle that applies only to a corporate upstream proxy.

Two operational gaps also exist:

- Brokers only send Telegram approval cards. One separate process must read
  the button clicks and route them to the owning broker. That process,
  `unyolo-telegram`, is not in the release bundle, and guided setup does not
  install it. A host without it shows approval cards whose buttons do nothing.
- A host that still runs the older release layout can lose its broker binaries
  during a partial migration while the services keep running from deleted
  files. The next restart then fails.

## What stays

- Brokers run as native systemd services on the host and keep every provider
  credential. A broad provider credential is supported, because the broker
  narrows each request: policy, grants, and approvals decide every call, and an
  installation credential is narrowed to the target repository and minimum
  permissions.
- One broker process per provider, each with its own listener, state, and
  audit stream.
- The GitHub-compatible API (`/api/v3` and `/api/graphql`) on the agent
  listener, with Git on the same host. Stock `gh` uses it through `GH_HOST`, and
  Git uses `gh auth git-credential`.

## Decisions

1. **A sandbox is a broker client, not an account.** Each sandbox gets its own
   client id, client secret, and policy rules. No Linux account is created for
   it.
2. **Sandboxes connect over TLS.** The agent listener listens on one address
   that the sandbox network can reach and nothing else, for example a private
   overlay network address. The sandbox's OpenShell policy allows only `gh` and
   `git` to reach that address and port.
3. **The client secret stays outside the sandbox.** An OpenShell provider holds
   the secret. The sandbox sees a placeholder in `GH_ENTERPRISE_TOKEN`, and the
   proxy replaces it on requests to the broker endpoint only.
4. **The broker certificate is publicly trusted.** Decision 3 requires it,
   because the proxy verifies the broker against public roots. With a private
   CA, the endpoint must use `tls: skip`. The proxy then cannot inject the
   secret, so the secret must sit in the sandbox, and every client must also
   receive the CA through `SSL_CERT_FILE` and `GIT_SSL_CAINFO`. Setup supports
   this fallback only when the operator chooses it explicitly.
5. **One Telegram reader per bot.** `unyolo-telegram` is the only process that
   reads the bot's updates. It routes button clicks to each broker over the
   broker's Operator V1 socket. Brokers keep sending cards and never read
   updates.
6. **No `sudo-broker` for sandbox agents.** It runs host commands as root for a
   local agent account. A sandbox agent has no such account and must not run
   host commands. Setup refuses to connect `sudo-broker` to a sandbox agent.

## Work

### 1. Ship the Telegram reader

- Add `unyolo-telegram` and its systemd unit to the release bundle and the
  release checksums and SBOM.
- Add a Telegram step to `unyolo setup`: bot token file, chat id, and the
  route of each installed broker. Setup takes each operator secret from the
  broker it installs, never from a command-line value.
- Setup installs `unyolo-telegram.service`, adds its account to the operator
  socket group of each broker, and checks every route before it reports
  success.
- A host can run only one reader per bot. Setup refuses to install a second
  reader for the same bot token, and the doctor reports any other running
  reader.

### 2. Add a sandbox agent target to setup

- Add the agent location `openshell` to the setup intent. Its fields are the
  connection name, the broker address the sandbox can reach, and the
  certificate mode (`public` or `private-ca`).
- For this target, setup:
  - creates the client id and secret slot and a policy rule set for the
    client, starting from the request-everything preset;
  - configures the agent listener as `tls://` on the chosen address with
    network exposure allowed, and refuses a wildcard bind address;
  - writes an OpenShell provider profile: one credential bound to
    `GH_ENTERPRISE_TOKEN`, endpoints for the broker address and port with
    REST and GraphQL inspection, and binaries `gh` and `git`;
  - writes the sandbox network policy fragment and the sandbox environment:
    `GH_HOST` and the Git `insteadOf` rule for `https://github.com/`;
  - prints the `openshell provider create` and `openshell sandbox create`
    arguments that use these files. The secret goes into the provider from a
    file and is never printed.
- The `private-ca` mode writes `tls: skip` and the CA file instead, and states
  that the client secret will be readable inside the sandbox.
- A sandbox target creates no Linux account and no client file in a home
  directory.

### 3. Certificates for the agent listener

- Keep operator-supplied certificate and key files as the only certificate
  source in the broker.
- Setup accepts a certificate from a public CA, for example an overlay network
  HTTPS certificate or an ACME certificate, and checks that the chain verifies
  against public roots for the configured name.
- The doctor warns 30 days before the certificate expires and fails after it
  expires.

### 4. Release-layout migration

- Treat a host with the older release layout as a fresh-state cutover: setup
  installs every selected broker from one release, stops and removes the old
  units and reader only after the new ones are installed and healthy, and
  never deletes a binary that a running unit still uses.
- `unyolo setup` and `unyolo repair` must refuse to remove provider files that
  belong to brokers outside the current session, instead of reconciling them
  as legacy.
- Discarded or abandoned sessions leave the host unchanged.

### 5. Hugging Face

- Out of scope for this plan. A later plan adds a compatible endpoint for the
  Hugging Face client in the same way, so a sandbox can use stock `hf`.

## Verification

An end-to-end run in a real OpenShell sandbox, with a broker installed by
`unyolo setup` and nothing installed by hand:

- `gh api`, `gh pr list`, `gh pr view --json`, `gh pr checks`, and
  `gh api --paginate` return GitHub data.
- `git clone` and `git push` of a branch work. The push waits for a Telegram
  approval, the button press changes the card, and the push completes.
- A repository without a rule returns `403`, and `curl` to the broker is
  blocked by the sandbox policy.
- No file, environment variable, or process argument in the sandbox contains
  the client secret in the default `public` certificate mode.
- Restarting each broker and the Telegram reader keeps pending approvals and
  button handling working.
- Unit and integration tests cover the new setup intent, the generated
  OpenShell files, the reader installation, and the refusal cases.

## Non-goals

- Running brokers inside OpenShell.
- Exposing the agent listener to the internet.
- Approving through a different chat application.
- Compatibility with the older release layout after the cutover.
