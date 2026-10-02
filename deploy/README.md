# Deploying the verifier backend

Two workflows. `docker-build.yml` publishes an image to GHCR on push.
`docker-deploy.yml` is manual only, so merging a branch never changes what is
running.

The deploy drives the box's Docker daemon over SSH. Nothing is copied to the
server: compose reads the file and the environment on the runner and sends the
daemon an already-expanded spec. The only things that exist on the box are the
container, the keystore directory, and the Docker socket.

## Before the first deploy

One thing. **The edge must be up:** nginx-proxy, acme-companion and the
`proxy-net` network are defined in `eudi-srv-wallet-provider`, not here. Deploy
that stack first. The preflight step checks and stops if `proxy-net` is missing.

Nothing has to be placed on the box by hand. The keystore is generated on first
deploy, as below.

## The access certificate keystore

`keystore-init` generates it into a named volume on first deploy and skips if it
already exists, the same shape as `eudi-srv-wallet-provider`'s `keystore-init`.
No manual step, nothing to `scp`, and the private key never passes through a CI
runner or a git repository.

Keeping it matters: the public key is published at
`/verifier/wallet/public-keys.json` and the wallet verifies every request object
against it. Regenerating invalidates anything already issued. That is why the
volume is named and why the init container is idempotent rather than
unconditional.

**It generates a two-level chain, not a self-signed certificate.** This looks
like overkill and is not. `AccessCertificate` (`VerifierConfig.kt:231`)
enforces:

    require(key.isPrivate)                              a private key is required for signing
    require(!key.parsedX509CertChain.isNullOrEmpty())   must have a non-empty certificate chain
    require(!certificate.isSelfSigned())                must not be self-signed

The third rejects a self-signed certificate outright, **whatever
`clientIdPrefix` is set to**. Found by running it: a self-signed leaf fails at
startup with `access certificate must not be self-signed`. So the init container
mints a throwaway CA and signs the leaf with it. Upstream's dev keystore is a
chain for the same reason.

That CA is a local convenience, not a trust anchor. Under `pre-registered` the
chain is never sent to the wallet, so nothing outside the container ever sees
it. If we move to `x509_san_dns` the chain does go to the wallet, and then it
has to come from `WEBUILD/pki/` instead.

**Do not use upstream's `src/main/resources/keystore.jks`.** It is committed to
every clone of this repository, so its private key is public. It is also P-521
with SANs for `localhost` and `verifier`.

Verified end to end before this was written: `keystore-init` generates a
P-256 key with `DNSName: demo.eudiw.grnet.gr`, skips on a second run, and the
service starts with it and signs a request object with `ES256`, publishing the
key at `/wallet/public-keys.json`.

## Repository secrets

| Secret | What it is |
| --- | --- |
| `SSH_KEY` | Private key authorised for `ubuntu@3.69.83.252`. Written to `~/.ssh/eudiw-deploy` on the runner. Paste the whole file, BEGIN and END lines included. |
| `VERIFIER_KEYSTORE_PASSWORD` | Any value you choose. `keystore-init` creates the keystore with it and the service opens it with it. Changing it after the first deploy makes the existing keystore unreadable. |
| `VERIFIER_REGISTRATION_CERTIFICATE` | ETSI 119 475 registration certificate, a signed JWT. Required: the service will not start without it. |

There is one keystore password, not two. A Java keystore has separate passwords
for the file and for each key entry, and the application does read them as
separate properties, but `keytool -importkeystore` gives the imported entry the
same password as the destination store. So both properties take the one secret.

Everything else is non-secret and committed in `stack.env`, where it is
reviewable in a diff.

## Routing: three paths on a shared hostname

This service is unusual in the stack. It exposes three top-level prefixes rather
than one, and a container can only declare `VIRTUAL_PATH` once. So it uses
nginx-proxy's `VIRTUAL_HOST_MULTIPORTS` instead:

    demo.eudiw.grnet.gr/verifier/ui           the UI backend API
    demo.eudiw.grnet.gr/verifier/utilities    validation helpers
    demo.eudiw.grnet.gr/verifier/wallet       what the phone talks to

`VIRTUAL_HOST_MULTIPORTS` **replaces** `VIRTUAL_HOST`, `VIRTUAL_PORT`,
`VIRTUAL_PROTO`, `VIRTUAL_PATH` and `VIRTUAL_DEST` on this container. Setting
any of them alongside it does nothing. `LETSENCRYPT_HOST` is separate and still
applies.

Despite the name, only one port is involved. It is being used for multiple
paths, not multiple ports.

Each path sets `dest` to strip the `/verifier` prefix, so `/verifier/wallet/x`
reaches the container as `/wallet/x`. The application serves `/ui`, `/wallet`
and `/utilities` at its root and knows nothing about the prefix, which is the
same arrangement as every other service here.

This was got wrong first time on the reasoning that the service builds absolute
URLs from `VERIFIER_PUBLICURL` and therefore needed the path preserved. Those
two are independent: the prefix in generated URLs comes from `publicUrl`, not
from the incoming request path. Leaving `dest` unset routes perfectly and then
404s inside the application, which reads as a routing fault and is not one. The
tell is in the nginx access log, which records the upstream it reached:

    "GET /verifier/wallet/public-keys.json" 404 "172.20.0.9:8080"

An unrouted path would never name an upstream.

Verified before writing this: nginx-proxy 1.11 is what runs on the box, and its
template references `VIRTUAL_HOST_MULTIPORTS`.

## Configuration

Spring's relaxed binding means every property has an environment variable form,
so the whole service is configured without a config file. `verifier.publicUrl`
becomes `VERIFIER_PUBLICURL`, and list entries take the
`VERIFIER_INTENDEDUSES[0]_ID` form.

| Variable | What it does |
| --- | --- |
| `VERIFIER_PUBLICURL` | Written into `request_uri` and `response_uri` inside the signed request object. The wallet resolves these, so it cannot be rewritten after issuance. |
| `VERIFIER_CLIENTIDPREFIX` | `pre-registered` here. See below. |
| `VERIFIER_ACCESS_CERTIFICATE_*` | The keystore holding the request-object signing key. |
| `VERIFIER_INTENDEDUSES[0]_*` | Why the verifier is asking. Required. |

`VERIFIER_PUBLICURL` deserves the same care as the status list's `SERVICE_URL`:
it ends up inside a signature, so changing it does not migrate anything already
issued.

Confirmed by running the image with a path-prefixed public URL: both
`request_uri` and `response_uri` come out carrying `/verifier`, so the prefix
survives Spring's URI building and no `SCRIPT_NAME` equivalent is needed.

## The client id prefix, and what the keystore has to be

`pre-registered` is the upstream default and what this stack uses.

Under `pre-registered` the signed request object carries only a `kid`
(`CreateJarNimbus.kt:90`). The certificate is never sent; the wallet is expected
to know the verifier out of band and reads the key from
`/verifier/wallet/public-keys.json`. So the keystore only has to hold a usable
EC key. Nothing validates a chain and nothing checks a SAN.

Under `x509_san_dns` or `x509_hash` the certificate chain goes into the JWS
header and the wallet validates it. That needs a leaf issued from
`WEBUILD/pki/` with `demo.eudiw.grnet.gr` as a SAN. Switching is a PKI ceremony
plus a `stack.env` change, not a redesign.

`okeanos-v6` carried a commit appending `"Verifier"` to
`DEFAULT_CLIENT_ID_PREFIXES_SUPPORTED`, an OpenID4VP spec constant. That was a
workaround for this setting and is not carried here.

**The algorithm must match the key's curve.** P-256 with `ES256`, P-521 with
`ES512`. A mismatch does not fail at startup; it fails at the first request with
`The ES256 algorithm is not allowed or supported by the JWS signer`.

## No healthcheck in the container

The Paketo runtime image has no shell and no HTTP client: `/bin/sh`,
`/bin/bash`, `/usr/bin/curl` and `/bin/busybox` are all absent. Every form of
compose `test:` would fail, so the service declares none. The wallet provider's
jib-built image is the same.

The service does expose Spring Boot actuator at `/actuator/health`, returning
`{"status":"UP"}`. The deploy workflow probes through the proxy instead, which
tests more: it exercises the vhost rather than just the process.

## What the verify step checks

- the container is running and attached to `proxy-net`
- `nginx -t` passes
- all three prefixes route: `/ui/intended-uses`, `/utilities/attestationClassifications`
  and `/wallet/public-keys.json` each return 200 through the proxy
- `/wallet-provider/jwks` and `/token_status_list/swagger/` still return 200, so
  adding three paths did not shadow the five services already on this hostname

The three-prefix check matters because they are three independent entries in
`VIRTUAL_HOST_MULTIPORTS`. One working says nothing about the others.

Routes that return 404 are deliberately not used as probes: nginx returns 404
when nothing is routed, so a 404 cannot distinguish a working route from a
broken one.

## Starting the JVM takes time

About 8 seconds before anything answers, longer on a cold pull. Both the
container-state loop and the first HTTP probe retry for 90 seconds. A probe
immediately after `up -d` would be testing the startup delay rather than the
deployment.
