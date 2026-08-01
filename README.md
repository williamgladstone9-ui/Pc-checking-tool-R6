R6 Verification Tool - Consent-First Open Source MVP
Version 0.1.0-MVP - Local-first, per-step authorized, no credential collection
> Reference implementation of the privacy-respecting design spec at `/home/user/CONSENT_FIRST_VERIFICATION_TOOL_DESIGN.md`
What This Is
A transparent verification tool for competitive Rainbow Six Siege communities. It runs entirely locally, asks for separate consent before every invasive step, minimizes data to IOC domains/hashes only, and produces a signed JSON + HTML report that the user previews, redacts, and explicitly approves before any export or upload.
It is NOT spyware:
No passwords, Discord tokens, browser cookies, payment data reading
No covert screenshots (only captures preview of findings already displayed inside app, with redaction)
No automatic uploads (MVP has upload disabled by code)
No Discord DM reading (violates ToS - explicitly not implemented)
No credential dumping, no kernel tampering, no AV bypass
Quick Start (Linux demo, Windows build same code)
```bash
# prerequisites: Rust stable 1.77+ (installed via rustup)
cargo build --release
./target/release/verify-ui wizard        # interactive wizard
./target/release/verify-ui demo-auto     # non-interactive demo using test_data
./target/release/verify-ui list-browsers # enumerate browser profiles safely
```
Windows build:
```
cargo build --release --target x86_64-pc-windows-msvc
# Binary: target\x86_64-pc-windows-msvc\release\verify-ui.exe
# Requires code signing with EV cert in production
```
Demo test data includes a fake Chrome History with 2 vendor URLs that match `ioc_packs/r6-sample.json`.
Running `demo-auto` produces:
`demo_report.json` (signed, machine-readable)
`demo_report.html` (human-readable)
Both show exactly what would leave device.
Architecture
```
verify-core (no OS deps, no network)
  - IocPack loader + validation
  - Finding, ConsentReceipt, FinalReport types
  - Redaction helpers, Ed25519 signing

verify-scanner (privileged logic, no network)
  - Trait ScanModule { disclosure(), enumerate_scopes(), scan() }
  - filesystem: walkdir max depth 3, hash only exe/dll candidates, name filter for minimization
  - process: sysinfo crate, name only, no memory dump
  - recyclebin: $I file parser (FILETIME -> ISO8601), $R hash, original path, deletion time
  - registry: Windows only via winreg, read-only Uninstall/Run/Services, offline hive stub
  - browser: profile discovery via Local State / profiles.ini, copy to temp for locked handling, rusqlite readonly immutable flag, queries ONLY for IOC domains

verify-ui (unprivileged UI)
  - CLI wizard using inquire/dialoguer (Tauri/Egui UI planned for Phase 2)
  - Intro -> Choose modules (all off) -> Disclosure -> Select profiles -> Authorize button -> Scan -> Review (include/redact/exclude/dispute + screenshot preview of app canvas only) -> Final preview + hash -> Export auth (local) -> Upload auth (disabled in MVP, shows what would be sent)
  - Vault: data_local_dir/R6Verification/vault, AES-GCM+DPAPI planned (currently file with auto-delete)
```
Privilege separation: UI never elevated. Scanner elevation requested only after "Authorize and run this step" and only for modules that need it (Recycle Bin, Registry HKLM). On Linux demo, those modules gracefully report NotScanned.
Per-Step Authorization Enforced
Every module has `Disclosure` struct rendered before scan
User must checkbox "I understand" then press distinct "Authorize and run this step"
System generates `ConsentReceipt { moduleId, timestamp, iocVersion, scopeHash, decision: Authorized, elevationRequested }`
Scanner validates receipt
Skip = `NotScanned { reason }` recorded as verification incomplete, not evidence
Final export requires separate confirmation, shows destination, hash
No "I agree to all" button.
IOC Packs
Signed declarative JSON in `ioc_packs/`:
```json
{
  "id": "r6-vendor-sample-2026-07-01",
  "domains": ["example-cheat-vendor1.com"],
  "file_hashes": [{"sha256": "...", "file_name": "loader.exe"}],
  "file_name_indicators": ["r6loader.exe"]
}
```
Load + validate, hash set for fast match, domain matching via host equality or substring + subdomain handling, no regex wildcards.
Production: sign with Ed25519, community review.
Report Format
JSON:
reportId, generatedAt, app version/buildHash/codeSignature
operator, iocPack id/version/hash
consent[] receipts
scanSummary { modulesRun, modulesSkipped, notScannedReason, limitations }
findings[] with observedFact vs inference vs disclaimer, benignExplanations, redacted paths, confidence
signature ed25519
HTML: human readable, badges for confidence, collapsible, audit log, preview exact.
Email & Discord - Why Not Implemented in MVP
As specified in design doc:
Discord: No official API for personal DM search. Token scraping violates ToS & CFAA. Only allowed path: user-selected Discord Data Package export (Settings->Privacy->Request Data). MVP shows placeholder and explains. Will implement user-selected export parser in Phase 2.
Email: Requires OAuth with Google/Microsoft verification, sensitive scopes, privacy policy. Implemented in Phase 2 as: provider-side search `q=from:vendor`, fetch headers first, second explicit auth for body, immediate revocation. Code present in design but not in MVP binary to avoid accidental credential handling.
Security Hardening
No network in scanner (only UI update check allowlisted)
Parser workers planned for AppContainer/job object sandboxing (currently same process but memory-limited, read-only SQLite)
Path traversal protection, max file size 100MB, YARA timeout 1s
SBOM: `cargo cyclonedx` or `cargo audit`
Reproducible build: `Cargo.lock` pinned, build hash in report
Code signing: Windows EV via Azure Trusted Signing, macOS notarization in Phase 3
Data Minimization in Code
See `crates/verify-scanner/src/filesystem.rs`:
```rust
// Only hash if name matches indicator or exe in Downloads with filter
let should_hash = name_match || (file_name.ends_with(".exe") && path.contains("Download"));
```
And `browser.rs`: `SELECT url,title,last_visit_time FROM urls WHERE ...` NOT `SELECT *`. Matched only.
Never read:
`Login Data`, `Cookies`, `Web Data`, `Local Storage/leveldb` (Discord tokens), `NTUSER.DAT` SAM, `*.ost` decryption.
Limitations ( Shown in UI )
Cannot prove non-cheating, fileless/DMA cheats invisible
AV may quarantine files
Encrypted Firefox profiles with Primary Password = NotScanned
Recycle Bin emptied = no data
Single URL/file != proof of purchase/use, ads cause FP, research visits identical
Battlesye/EAC kernel may block scanning
Roadmap
MVP (this): Windows/Linux, filesystem/process/recyclebin/registry/browser, local reports, per-step auth.
Phase 2: Gmail/Graph OAuth narrow search, PST/mbox/EML user-selected, Discord data package, improved correlation timeline, optional encrypted upload with pinned cert, audit by third party.
Phase 3: macOS Safari + launchd, additional games IOC packs, legal review for minors/GDPR, Tauri/Egui hardened UI.
Legal - Not Legal Advice
See design doc Section 4. Requires counsel for GDPR, CCPA, ECPA/SCA, CFAA, employment, minors. Provide plain-language consents. Refusal != cheating.
Running the Wizard
```
$ ./target/release/verify-ui wizard
====================================================
 R6 Verification Tool - Consent-First MVP v0.1.0
...
Select modules to run (all OFF by default).
> [ ] filesystem - File & System Checks
  [ ] processes - Running Processes...
...
=== MODULE: Filesystem ... ===
Source: Filesystem ...
Fields read: File name, Path...
...
>> Authorize and run this step: File & System Checks ? y/n
...
Review findings ...
Decision for F-... [Include as-is etc]
...
Final preview hash: abc...
Authorize local export ? y/n
```
Verification of Build
```bash
sha256sum target/release/verify-ui
# Compare to builds.txt signed in repo
```
License
Recommended MIT OR Apache-2.0 for code (allows audit), CC-BY-4.0 for docs. Add LICENSE file before public release.
Contributing
All contributions must:
Not add credential collection (CI regex blocks discord.*token, Login Data, etc)
Add disclosure text for new module
Provide sample IOC pack change with justification
Include test with redaction
---
Built with care to be auditable, not maximally collecting. If a feature would require bypassing security or breaking ToS, it is deliberately omitted.
