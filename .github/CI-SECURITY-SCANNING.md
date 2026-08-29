# CI Container Image Scanning

**Purpose:** How the `Build & Push` workflow scans service images for
vulnerabilities, what blocks a release, and how to review or allowlist
findings. (PETPLAT-105.)

## Where it runs

`.github/workflows/reusable/build-scan-push.yml`, once per changed service:

```
build JAR → build linux/arm64 image (--load, NOT pushed)
          → Trivy scan: CRITICAL  → exit 1 → job fails → image never reaches ECR
          → Trivy scan: HIGH+CRIT → JSON report (exit 0)
          → warn if any HIGH
          → upload trivy-results/ artifact
          → docker push  (only if the CRITICAL gate passed)
```

The scan gates the push — a failing image is never published. This is *in
addition* to ECR's own scan-on-push.

## What blocks vs. warns

| Severity | Effect |
|----------|--------|
| `CRITICAL` | **Blocks.** Job fails, no push, `dispatch` job does not run, so no deploy. |
| `HIGH` | Warns only. A `::warning` annotation on the run + listed in the artifact. |
| `MEDIUM` / `LOW` / `UNKNOWN` | Not reported by CI (view locally if needed). |
| `unfixed` | Included (`ignore-unfixed: false`) — an unfixed CRITICAL still blocks; allowlist it if accepted. |

## Reviewing scan results

1. Open the failed (or succeeded) run in the **Actions** tab.
2. The **CRITICAL** step's log shows the blocking CVEs in a table.
3. Download the **`trivy-<service>-<sha>`** artifact for the full HIGH+CRITICAL
   JSON (one file per service). Pretty-print a summary:
   ```bash
   jq -r '.Results[]?.Vulnerabilities[]? | [.Severity,.VulnerabilityID,.PkgName,.InstalledVersion,.FixedVersion] | @tsv' \
     trivy-results/<service>.json | sort
   ```

## Fixing

Preferred, in order:
1. **Bump the dependency** — most CVEs are in a transitive library; update the
   version in the service's `pom.xml` (or the parent) and re-push.
2. **Rebuild the base image** — `eclipse-temurin:17` CVEs clear when a newer
   patch tag is pulled; the `Dockerfile` `FROM` pins only the major, so a
   fresh build usually picks up fixes.
3. **Allowlist** — only when there is no fix and the CVE is not reachable in
   our usage.

## Allowlisting an accepted CVE

Edit **`.trivyignore`** at the repo root. One CVE ID per line, with a comment:

```
CVE-2024-99999   # transitive in netty, no fix; not exploitable (no untrusted input path) — accepted by <name>, 2026-08-29, re-review 2026-11-29
```

Trivy reads `.trivyignore` automatically and the workflow also passes it
explicitly. Keep the list short and time-boxed — every entry needs a reason,
an owner, and a re-review date.

## Running the scan locally

```bash
SVC=customers-service
mvn -q -pl spring-petclinic-$SVC -am package -DskipTests
JAR=$(basename "$(ls spring-petclinic-$SVC/target/spring-petclinic-*.jar | grep -v -- -plain)" .jar)
docker buildx build --platform linux/arm64 --provenance=false \
  --build-arg ARTIFACT_NAME="$JAR" --build-arg EXPOSED_PORT=8081 \
  -f docker/Dockerfile -t local/$SVC:scan --load spring-petclinic-$SVC/target
trivy image --severity HIGH,CRITICAL local/$SVC:scan
```
