# Supply chain and deployment

Reference for reviewing what goes into a build, what is allowed to run, and how security changes reach production.

---

## 1. Dependencies

### 1.1 Choose maintained dependencies

Prefer popular, actively maintained crates; avoid ones with a history of unfixed or slowly fixed vulnerabilities, especially for cryptography, parsing, and authentication. Evaluate a new dependency before adopting it.

### 1.2 Pin every input

Every build input is unambiguous — a resolved version or a hash. Commit `Cargo.lock` for binaries; pin git dependencies to a `rev`, not a branch; pin base images by digest, not `latest`; do not fetch inputs over plain HTTP at build time. Remember that build scripts (`build.rs`) and procedural macros run arbitrary code during the build.

### 1.3 Stay current, rebuild often

Patches reach production only when you rebuild and redeploy. Track advisories (`cargo audit`, `cargo deny check advisories`) in CI, upgrade regularly so an urgent patch is a small change, and rebuild images often.

### 1.4 Outside code is reviewed

Externally contributed code, vendored code, and automated dependency bumps are reviewed and tested like any change.

---

## 2. Builds and deployment

### 2.1 Every build records its provenance

Each artifact records where it came from: source commit, dependencies, build command, and builder. Deploy by digest, not by mutable tag.

### 2.2 Check what is deployed, not only who deployed it

An authorized person can make a mistake or be malicious. Deployment gates verify the artifact: provenance is authentic and matches the exact artifact (by hash), and policy allows it. Log enough at the decision point to re-check the decision later.

- **Flag:** a deploy gate that is only role-based access for the deployer; images accepted from any registry or tag.

### 2.3 Only recently built, immutable images

Patch by rebuilding and redeploying, never by changing running containers. Enforce a maximum image age so old, unpatched images are not redeployed.

### 2.4 Configuration and access rules as reviewed code

Configuration and cloud resource access rules (bucket policies, role bindings) are version-controlled, reviewed, and changed only by the deployment pipeline — humans do not edit them directly in production. Ideally targets verify a signature on config that the push automation itself cannot forge. Validate dangerous values automatically (an empty list, a wildcard, a root path).

### 2.5 Bypasses alert

Any deployment that skips policy (`--force`, `--skip-checks`) raises an alarm, is audited quickly, and stays rare.

---

## 3. Rolling out security changes

- **Small and standalone.** A security change is not bundled with refactors or unrelated improvements; small changes are easier to review and roll back.
- **Tested and reviewed**, including the denied cases.
- **Behind a flag that changes nothing when off.** With the flag off, the binary behaves exactly as before.
- **Dry run first.** New authorization or validation rules run in log-only mode — counting would-be denials — before enforcement, to find callers that are wrongly in scope.
- **Gradual, with canary metrics** that show behavior before and after.
- **Emergency patches still roll out gradually**, with the same tests and canaries, just faster.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://rustsec.org/ — RustSec advisory database (`cargo audit`)
- https://embarkstudios.github.io/cargo-deny/ — cargo-deny
