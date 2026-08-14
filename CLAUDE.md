# CLAUDE.md

Notes for working on this repo, gathered while adopting CI/CD (build matrix, Keycloak-version
tests, CodeQL, release automation). These aren't derivable from the code alone.

## What this plugin actually is, and why that bounds everything else

This plugin exists only because Keycloak **< 20.0.0** mishandles ELSTER's custom SAML attribute
types (`ekona:*`); the bug was fixed upstream in 20.0.0 (see README). It deploys via the
**legacy WildFly distribution** (`wildfly-maven-plugin`, `standalone/deployments`,
`jboss-deployment-structure.xml`), which Keycloak deprecated in 17 and removed entirely in 21.
So any meaningful "test against multiple Keycloak versions" work is bounded to roughly the 15–19
range — testing against current Keycloak (24+, Quarkus distribution) tests nothing this plugin
does.

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

## Test strategy

`CustomSAMLEndpoint`'s core value — rewriting `ekona:*` SAML attribute types to `xsd:string` and
flattening composite values — is pure `org.w3c.dom` manipulation with no Keycloak server
dependency (`replaceUnknownType` and its private helpers). That's what
`CustomSAMLEndpointTest` covers. A true end-to-end ELSTER SAML login test isn't feasible in CI
(needs real IdP metadata, signing certs, encrypted assertions); the `end2end` build.yml job is
deliberately scoped as a deploy/boot smoke test only, not functional SAML coverage.
