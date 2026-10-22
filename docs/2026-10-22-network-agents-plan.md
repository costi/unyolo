---
date: 2026-10-22
author: Onur Solmaz <osolmaz@users.noreply.github.com>
title: Serve network agents in sandboxes, containers, and other computers
tags: [setup, agents, sandbox, openshell, docker, telegram, github, tls]
---

# Serve network agents in sandboxes, containers, and other computers

## Goal

An agent that does not run as a local account on the broker host uses stock
`gh` and `git` against GitHub through gh-broker. This covers sandboxes such as
OpenShell, Docker containers on the same host, and other computers. The agent
holds no provider credential. Approvals arrive in Telegram, and the buttons
work for every broker. One `unyolo setup` run installs and connects all of it,
with no manual host steps.

## Why the current setup does not fit

The shipped setup supports one agent type: a local Linux account.

- The agent proves its identity with a local Unix socket, guarded by a group,
  plus a client secret in its home directory.
- `sudo-broker` runs approved privileged commands as that agent's Linux user.
- The setup intent already names `container` and `remote` agents, but neither
  ships, and they are designed as separate paths.

An agent in a sandbox, a container, or on another computer has no host account
and cannot use host Unix sockets. For the broker, all three are the same thing:
a client that connects over the network with a secret. They differ only in how
the secret reaches the agent and how the runtime restricts the agent's network.

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
- Local account agents keep the Unix socket path unchanged.
- The GitHub-compatible API (`/api/v3` and `/api/graphql`) on the agent
  listener, with Git on the same host. Stock `gh` uses it through `GH_HOST`, and
  Git uses `gh auth git-credential`.

## Decisions

1. **A network agent is a broker client, not an account.** Each agent gets its
   own client id, client secret, and policy rules. No Linux account is created
   for it.
2. **Network agents connect over TLS.** The agent listener listens on one
   address that the agents can reach and nothing else, for example a private
   overlay network address or a container bridge address. There is no plain
   TCP and no wildcard bind address.
3. **One core, one adapter per runtime.** The setup core creates the client,
   its secret and policy, and the TLS listener. A runtime adapter turns that
   into the files and commands for one runtime. Brokers contain no
   runtime-specific code.
4. **Each adapter declares where the client secret lives:**
   - `injected`: the runtime adds the secret outside the agent, so code in the
     agent cannot read it. OpenShell does this with a provider placeholder.
   - `in_agent`: the agent can read the secret, from a mounted file or from
     its own credential store.

   Setup states which case applies before it asks for approval.
5. **The broker certificate is publicly trusted by default.** Every adapter
   works with it, and `injected` adapters whose runtime re-encrypts TLS need
   it: OpenShell verifies the upstream against public roots. A private CA is
   allowed when the operator chooses it. Then every agent receives the CA file,
   and an OpenShell agent falls back to `in_agent` with `tls: skip` on the
   broker endpoint.
6. **One Telegram reader per bot.** `unyolo-telegram` is the only process that
   reads the bot's updates. It routes button clicks to each broker over the
   broker's Operator V1 socket. Brokers keep sending cards and never read
   updates.
7. **No `sudo-broker` for network agents.** It runs host commands as root for a
   local agent account. A network agent has no such account and must not run
   host commands. Setup refuses to connect `sudo-broker` to a network agent.

## Adapters

| Adapter | Runtime | Secret | Network restriction | Enrollment |
| --- | --- | --- | --- | --- |
| `openshell` | OpenShell sandbox | `injected` | OpenShell network policy | Provider created from a file |
| `docker` | Docker container on the broker host | `in_agent` | Container network | Secret file mounted read-only |
| `remote` | Another computer | `in_agent` | The other computer's firewall | One-use pairing, as designed in `GUIDED_INSTALLATION.md` |

The adapter contract is the same for all three:

- **Input:** connection name, client id, secret slot, broker address, server
  name, certificate mode, and the providers to connect.
- **Output:** the runtime files, the commands for the operator to run, and the
  declared secret placement. A secret is written only to a file with
  owner-only access, or sent through the enrollment flow, and is never
  printed.
- **Agent environment:** `GH_HOST`, the Git `insteadOf` rule for
  `https://github.com/`, `gh auth git-credential` as the Git credential
  helper, and the CA file in private CA mode.

This plan builds the `openshell` adapter first. `docker` and `remote` move onto
the shared core when they ship, and keep their existing designs for secret
delivery and enrollment.

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

### 2. Add the network agent core to setup

- Replace the separate `container` and `remote` locations in the setup intent
  with one `network` location that has a `runtime` field (`openshell`,
  `docker`, or `remote`). This is an in-place change to setup intent v1.
- For a network agent, setup:
  - creates the client id, the secret slot, and a policy rule set for the
    client, starting from the request-everything preset;
  - configures the agent listener as `tls://` on the chosen address with
    network exposure allowed, and refuses a wildcard bind address;
  - calls the adapter and shows its declared secret placement in the review
    step.
- A network agent creates no Linux account and no client file in a home
  directory on the broker host.

### 3. Build the OpenShell adapter

- Write an OpenShell provider profile: one credential bound to
  `GH_ENTERPRISE_TOKEN`, endpoints for the broker address and port with REST
  and GraphQL inspection, and binaries `gh` and `git`.
- Write the sandbox network policy fragment and the sandbox environment.
- Print the `openshell provider create` and `openshell sandbox create`
  arguments that use these files. The secret goes into the provider from a
  file and is never printed.
- In private CA mode, write `tls: skip` and the CA file instead, and state that
  the client secret will be readable inside the sandbox.

### 4. Certificates for the agent listener

- Keep operator-supplied certificate and key files as the only certificate
  source in the broker.
- Setup accepts a certificate from a public CA, for example an overlay network
  HTTPS certificate or an ACME certificate, and checks that the chain verifies
  against public roots for the configured name.
- The doctor warns 30 days before the certificate expires and fails after it
  expires.

### 5. Release-layout migration

- Treat a host with the older release layout as a fresh-state cutover: setup
  installs every selected broker from one release, stops and removes the old
  units and reader only after the new ones are installed and healthy, and
  never deletes a binary that a running unit still uses.
- `unyolo setup` and `unyolo repair` must refuse to remove provider files that
  belong to brokers outside the current session, instead of reconciling them
  as legacy.
- Discarded or abandoned sessions leave the host unchanged.

### 6. Hugging Face

- Out of scope for this plan. A later plan adds a compatible endpoint for the
  Hugging Face client in the same way, so a network agent can use stock `hf`.

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
  the client secret in the default certificate mode.
- Restarting each broker and the Telegram reader keeps pending approvals and
  button handling working.
- Unit and integration tests cover the network intent, the adapter contract,
  the generated OpenShell files, the reader installation, and the refusal
  cases.

When the `docker` and `remote` adapters ship, the same end-to-end checks run
for them, except the secret check, which then confirms that the secret exists
only in the declared place.

## Non-goals

- Running brokers inside sandboxes or containers.
- Exposing the agent listener to the internet.
- Runtime-specific code in brokers.
- Adapters for other runtimes in this plan.
- Approving through a different chat application.
- Compatibility with the older release layout after the cutover.
