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

**A single Java source file can't import both `javax.ws.rs` and `jakarta.ws.rs`.** Since a
mechanical, single-tree upgrade wasn't possible, and the plugin needs to keep working on
pre-20 Keycloak (the whole reason it exists), the fix is dual source roots behind a Maven
profile pair, chosen at build time:

- `src/main/java-legacy/` — the original javax-based code, targeting Keycloak ≤ ~20 (WildFly).
- `src/main/java-jakarta/` — the ported jakarta-based code, targeting current Keycloak (Quarkus).
- `src/main/java/` — `ElsterIdentityProviderFactory.java` and `ElsterUserAttributeMapper.java`
  stay here, shared and unmodified; verified they compile unchanged at every version 15.1.1
  through 26.7.1.

Selected via `-Plegacy` (default, `activeByDefault=true`) or `-Pjakarta`, using
`build-helper-maven-plugin`'s `add-source` goal — an explicit `-P` on the command line
automatically deactivates the `activeByDefault` profile, so the two are mutually exclusive
without extra wiring.

**Only two files needed forking**: `CustomSAMLEndpoint.java` and `ElsterIdentityProvider.java`.
Everything else was unaffected — confirmed by compiling against 26.7.1 and checking which files
actually errored (only `CustomSAMLEndpoint.java`, even though `ElsterIdentityProvider.java`
*calls* the now-incompatible constructor — Java only flags the file whose own body breaks, not
every caller of a still-syntactically-valid method signature).

Exact port, in case this needs redoing for a future API break:

- `XMLEncryptionUtil.DecryptionKeyLocator` is trivial (`List<PrivateKey> getKeys(EncryptedData)`)
  — wrap the existing `PrivateKey` as `encryptedData -> Collections.singletonList(privateKey)`.
- Current `AssertionUtil.decryptAssertion(ResponseType, DecryptionKeyLocator)` internally is
  almost exactly what this plugin's forked `decryptAssertion` already does — same
  decrypt-into-temp-`Document` → `SAMLParser.parse` → `responseType.replaceAssertion(...)` shape,
  just with `replaceUnknownType(...)` spliced in before the parse, same as before.
- **Dropped** the manual "EncryptedID → BaseID" block from `handleLoginResponse` in the jakarta
  variant: current `SAMLEndpoint.handleLoginResponse` already calls `AssertionUtil.decryptId(...)`
  internally, so the plugin's old workaround is now redundant (and would only be a silent no-op
  if kept, since it null-checks before acting — but simpler to just remove it).
- **Kept** the `config.setWantAssertionsEncrypted(false)` / restore trick around
  `super.handleLoginResponse(...)` — still required, still works the same way (same guard clause
  in current code: `if (config.isWantAssertionsEncrypted() && !assertionIsEncrypted)` → error
  page, and our pre-decryption still flips `assertionIsEncrypted` to `false` before super runs).
- `postBinding()` needed an added `samlArt` `@FormParam` and a 5th arg to `execute(...)` — the
  base class's own `postBinding()` signature grew an artifact-binding parameter upstream.
- `ElsterIdentityProvider.callback(RealmModel realm, AuthenticationCallback callback, EventBuilder event)`
  keeps the exact same signature in current Keycloak, but its body must pass the inherited
  `session` field (from `AbstractIdentityProvider`, present in both API generations) instead of
  the `realm` parameter, since jakarta's `CustomSAMLEndpoint` constructor takes `KeycloakSession`
  first. That's the one line that forces this file to be forked too.
- `replaceUnknownType` and all its private helpers are copied byte-for-byte between variants —
  they only touch `org.w3c.dom`/`javax.xml.xpath` (JDK standard library, unrelated to the
  Jakarta EE rename).

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
