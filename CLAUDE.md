# CLAUDE.md

Notes for working on this repo, gathered while adopting CI/CD (build matrix, Keycloak-version
tests, CodeQL, release automation). These aren't derivable from the code alone.

## What this plugin actually is, and why that bounds everything else

This plugin exists only because Keycloak **< 20.0.0** mishandles ELSTER's custom SAML attribute
types (`ekona:*`); the bug was fixed upstream in 20.0.0 (see README). It originally deployed only
via the **legacy WildFly distribution** (`wildfly-maven-plugin`, `standalone/deployments`,
`jboss-deployment-structure.xml`), which Keycloak deprecated in 17 and removed entirely in 21.
**As of the Jakarta/Quarkus migration below, it now also supports current Keycloak** (Quarkus
distribution, `providers/` dir), built via a separate Maven profile — see "Supporting both old
and current Keycloak" for how that's structured. The `replaceUnknownType`/flattening logic still
has value on current Keycloak even though the original parsing bug is fixed there: it's the thing
that turns nested ELSTER attributes (e.g. address) into separate flat user attributes, which
upstream Keycloak has no reason to replicate.

## Keycloak legacy-distribution version matrix gotchas

- **Not every version is on Maven Central.** E.g. `org.keycloak:keycloak-core:18.0.3` doesn't
  exist (404) even though 18.0.0/18.0.1/18.0.2 do. Always verify with a HEAD request against
  `https://repo.maven.apache.org/maven2/org/keycloak/<artifact>/<version>/` before pinning a
  version in a matrix — don't assume patch releases are uniformly present.
- **Docker `-legacy` tags on `quay.io/keycloak/keycloak` only exist for 17.0.0 through 19.0.3.**
  Versions before 17 use the plain tag with no suffix (e.g. `15.1.1`, since there was only one
  distribution then). There is **no legacy Docker image for 20.x** at all, even though the
  20.0.x legacy distribution zip itself exists — the images simply stopped being published after
  19.0.3. Check `https://quay.io/api/v1/repository/keycloak/keycloak/tag/?filter_tag_name=like:legacy`
  before assuming an image exists.
- Current working matrix (build.yml `keycloak-version-matrix` / `end2end`): **15.1.1, 18.0.2,
  19.0.3**.

## The legacy Keycloak Docker images hit a JDK cgroup-v2 bug on GitHub Actions

All three legacy images (`15.1.1`, `18.0.2-legacy`, `19.0.3-legacy`) are built
`FROM registry.access.redhat.com/ubi8-minimal` with `microdnf install java-11-openjdk-headless`
(see `keycloak/keycloak-containers` repo, tag-per-version, `server/Dockerfile`). The JDK build
baked in is frozen at whatever was current in the UBI8 repos when the image was published
(2021/2022). Old JDK 11 builds hit **JDK-8287073** ("NPE from `CgroupV2Subsystem.getInstance()`")
on cgroup-v2 hosts — which is what GitHub's `ubuntu-latest` runners use. Symptom: the container
crashes on boot with
`Exception in thread "main" java.lang.NullPointerException at ... CgroupV2Subsystem.getInstance ... at org.jboss.modules.Main.main`,
before Keycloak/WildFly code ever runs.

**What does NOT fix it** (tried, confirmed ineffective):
- `-e JAVA_OPTS_APPEND=-XX:-UseContainerSupport` — the JDK builds
  `com.sun.management.OperatingSystemMXBean` unconditionally, independent of that flag.
- `docker run --cgroupns=host` — doesn't change the outcome either.

**What fixes it**: rebuild a derived image that upgrades the JDK before deploying the plugin —
UBI8 is a no-subscription-required base, so its repos are reachable directly:
```dockerfile
FROM <legacy-image>
USER root
RUN microdnf update -y java-11-openjdk-headless && microdnf clean all
USER 1000
COPY --chown=1000:0 target/*.jar /opt/jboss/keycloak/standalone/deployments/
```
See the `end2end` job in `.github/workflows/build.yml` for the working version.

## This sandbox can't verify any of this locally

This dev machine is arm64 (Apple Silicon). All these legacy Keycloak images are amd64-only, and
there's no QEMU binfmt registered, so both `docker run` and `docker build` (any `RUN` step that
executes a binary) fail with `exec format error`. Docker-based workflow changes for this repo
can only be verified by pushing and watching the actual GitHub Actions run (native amd64
runners) — don't spend time trying to reproduce container boot issues locally here.

## Release automation prerequisites (not yet provisioned)

`release.yml` (Maven Central publish via `central-publishing-maven-plugin` + GPG signing) is
wired up but will fail at the publish step until someone provisions:
- Verified ownership of the `de.muenchen.keycloak` namespace on the Central Publishing Portal
  (or the groupId changes to something the repo owner controls — this repo is a personal fork,
  `t12v/keycloak-elster-provider`, of Landeshauptstadt München's original project).
- Repo secrets: `OSS_SONATYPE_USER`, `OSS_SONATYPE_PASS`, `OSS_SONATYPE_GPG_PRIVATE_KEY`,
  `OSS_SONATYPE_GPG_PASSPHRASE`.

It's manual-dispatch only, so this doesn't block normal CI.

## Supporting both old and current Keycloak: the Jakarta/Quarkus migration

The source **doesn't compile past Keycloak 20.0.5** as originally written. Bisected precisely
against `org.keycloak:keycloak-core` on Maven Central: 20.0.5 compiles, 21.1.2 breaks
(`SAMLEndpoint`'s constructor changed from `RealmModel` to `KeycloakSession` as its first param;
`XMLEncryptionUtil`/`AssertionUtil` decrypt methods changed from raw `PrivateKey` to a
`DecryptionKeyLocator` functional interface), and 22.0.5+ additionally breaks because Keycloak's
Jakarta EE 10 migration renamed `javax.ws.rs.*` → `jakarta.ws.rs.*`. Diffed against
`keycloak/keycloak` on GitHub at tag `26.7.1` (`services/.../broker/saml/SAMLEndpoint.java`,
`saml-core/.../util/XMLEncryptionUtil.java`, `AssertionUtil.java`) to find the exact deltas.

**A single Java source file can't import both `javax.ws.rs` and `jakarta.ws.rs`** — and, it turns
out, the post-Jakarta era isn't one API shape either. The SAML broker SPI moved *again*, twice,
within the 26.x line alone, discovered when a `keycloak.version=25.0.6` matrix entry (added
alongside a `latest`/26.7.1 one) failed to compile against code that had only ever been tested at
26.7.1:

| Change | Landed at |
|---|---|
| `SAMLEndpoint` ctor `RealmModel` → `KeycloakSession`; decrypt `PrivateKey` → `DecryptionKeyLocator` | 21.1.2 |
| `javax.ws.rs.*` → `jakarta.ws.rs.*` | 22.0.5 |
| `postBinding()`/`execute()` gains a `samlArt` param (4→5 args) | 26.0.0 |
| `AuthenticationCallback` moves from `IdentityProvider` to `UserAuthenticationIdentityProvider` | 26.5.0 |
| `SAMLEndpoint.session` field goes `private` → `protected` | 26.6.0 |

So there are three mutually-incompatible shapes to build against, each behind its own Maven
profile (`build-helper-maven-plugin`'s `add-source`, one `activeByDefault`, the rest explicit
`-P`, which auto-deactivates the default):

- `src/main/java-legacy/` (`-Plegacy`, default) — javax, Keycloak ≤ 20.0.5, WildFly.
- `src/main/java-jakarta/` (`-Pjakarta`) — jakarta, Keycloak 22.0.5–25.x: 4-arg `execute()`,
  `IdentityProvider.AuthenticationCallback`, `session` needs re-shadowing (private in super).
- `src/main/java-jakarta-current/` (`-Pjakarta-current`) — jakarta, Keycloak ≥ 26.5.0: 5-arg
  `execute()`, `UserAuthenticationIdentityProvider.AuthenticationCallback`, `session` protected
  (still re-shadowed anyway, cheap insurance against it moving a third time).
- `src/main/java/` — `ElsterIdentityProviderFactory.java` and `ElsterUserAttributeMapper.java`
  stay here, shared and unmodified across all three; confirmed to compile unchanged at every
  version from 15.1.1 through 26.7.1.

**Known gap, not currently in the CI matrix**: Keycloak 21.x (has the ctor/decrypt change but
predates the jakarta rename) and 26.0.0–26.4.x (has the 5-arg `execute()` but predates the
`UserAuthenticationIdentityProvider`/protected-`session` move) fall between these three variants
and aren't covered by any of them. Not worth a fourth variant unless something actually needs to
target that narrow window.

**Only two files ever need forking**: `CustomSAMLEndpoint.java` and `ElsterIdentityProvider.java`
(the latter only because it constructs `CustomSAMLEndpoint`, whose constructor signature differs
per shape — its own signature and logic are otherwise unchanged across all three, byte-identical
files, differing from `-legacy`'s only in using `session`+`KeycloakSession` instead of
`realm`+`RealmModel`). Confirmed by compiling against 26.7.1 and checking which files actually
errored — Java only flags the file whose own body breaks, not every caller of a
still-syntactically-valid method signature.

Exact port notes, in case this needs redoing for a future API break:

- `XMLEncryptionUtil.DecryptionKeyLocator` is trivial (`List<PrivateKey> getKeys(EncryptedData)`)
  — wrap the existing `PrivateKey` as `encryptedData -> Collections.singletonList(privateKey)`.
- Current `AssertionUtil.decryptAssertion(ResponseType, DecryptionKeyLocator)` internally is
  almost exactly what this plugin's forked `decryptAssertion` already does — same
  decrypt-into-temp-`Document` → `SAMLParser.parse` → `responseType.replaceAssertion(...)` shape,
  just with `replaceUnknownType(...)` spliced in before the parse, same as before.
- **Dropped** the manual "EncryptedID → BaseID" block from `handleLoginResponse` in both jakarta
  variants: current `SAMLEndpoint.handleLoginResponse` already calls `AssertionUtil.decryptId(...)`
  internally, so the plugin's old workaround is now redundant.
- **Kept** the `config.setWantAssertionsEncrypted(false)` / restore trick around
  `super.handleLoginResponse(...)` — still required, still works the same way (same guard clause:
  `if (config.isWantAssertionsEncrypted() && !assertionIsEncrypted)` → error page, and our
  pre-decryption still flips `assertionIsEncrypted` to `false` before super runs).
- **Re-added** an explicit `@Context private KeycloakSession session;` field to both jakarta
  variants (same trick the original legacy code used, for the same reason: don't trust the
  superclass field's visibility — it's genuinely changed once already). Shadowing costs nothing
  and works regardless of whether the inherited field is private or protected.
- `postBinding()`/`execute()`: 4 args pre-26.0.0, 5 (adds `samlArt`) from 26.0.0 on — this is the
  one change that can't be papered over with a compatibility trick, hence the 3-way split.
- `replaceUnknownType` and all its private helpers are copied byte-for-byte across all three
  variants — they only touch `org.w3c.dom`/`javax.xml.xpath` (JDK standard library, untouched by
  any of the above).
- Each variant's jar gets a distinct Maven classifier (`jakarta`, `jakarta-current`; legacy stays
  unclassified/primary) via `maven-jar-plugin` config inside each profile, so all three can be
  built back-to-back from the same tag without overwriting each other — see `release.yml`.

Current Quarkus-based Keycloak deploys providers completely differently from WildFly: copy the
jar into `/opt/keycloak/providers/` and run `/opt/keycloak/bin/kc.sh build` (which instantiates
providers and fails the build outright if one is broken — a stronger CI signal than the legacy
job's log-polling), then run with `start-dev`. No `/auth` path prefix. See the `end2end-quarkus`
job in `.github/workflows/build.yml`.

## Test strategy

`CustomSAMLEndpoint`'s core value — rewriting `ekona:*` SAML attribute types to `xsd:string` and
flattening composite values — is pure `org.w3c.dom` manipulation with no Keycloak server
dependency (`replaceUnknownType` and its private helpers). That's what
`CustomSAMLEndpointTest` covers. A true end-to-end ELSTER SAML login test isn't feasible in CI
(needs real IdP metadata, signing certs, encrypted assertions); the `end2end` build.yml job is
deliberately scoped as a deploy/boot smoke test only, not functional SAML coverage.
