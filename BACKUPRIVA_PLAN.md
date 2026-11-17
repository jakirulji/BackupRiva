# BackupRiva — Product & Engineering Plan

**Tagline:** Your Data. Always Protected.  
**Archive format:** `.abx`  
**Initial platform:** Windows desktop + Android companion  
**Repository:** https://github.com/jakirulji/BackupRiva

## 1. Product vision

BackupRiva is a standalone, file-based backup and restore system. It is separate from Apex Backup. Windows is the primary controller and processing host; Android is a companion storage/transfer endpoint. The product targets personal files and folders, not full operating-system or app-private-state migration.

## 2. Core requirements

- Back up and restore files between PC↔PC, PC↔Android, Android↔Android, and Android↔PC.
- Preserve original directory hierarchy, filenames, timestamps where supported, and file bytes.
- Provide full and incremental backups, selective restore, archive verification, and clear progress.
- Produce a portable, self-contained `.abx` archive that can be restored without a separate database or a missing base archive.
- Resume interrupted operations after app exit, reboot, cable removal, or Wi-Fi interruption.
- Detect source changes during backup; never silently treat a changed file as the original snapshot.
- Stream large files with bounded memory and avoid loading entire files into RAM.

## 3. Archive and snapshot design

- Define a versioned archive format with a manifest, object/chunk records, integrity metadata, and a crash-safe finalization/index strategy.
- Archive metadata must be sufficient for independent restore and verification.
- Incremental snapshots must remain independently restorable: either reference content already embedded in the same archive or use a documented portable package strategy. Do not create an incremental archive that silently depends on an unavailable earlier file.
- Whole-file deduplication is mandatory. Evaluate content-defined chunking (CDC) for large-file deduplication after measuring complexity and restore performance.
- Preserve source paths and metadata; handle filename collisions and cross-platform-invalid names explicitly.

## 4. Compression and integrity

- Use Zstandard (Zstd); Balanced mode around level 3 is the initial default.
- Detect already-compressed formats and use raw storage when compression would not provide a meaningful benefit.
- Never use lossy transformations. Do not promise major savings for JPEG, PNG, MP4, MP3, ZIP, APK, and similar formats.
- Use SHA-256 for file/object integrity and verify restored output against recorded hashes.
- Include archive verification and clear reporting of corrupt or unreadable items.

## 5. Encryption and recovery

- Use established cryptographic libraries; do not invent custom cryptography.
- Target AES-256-GCM for authenticated encryption and Argon2id for password-based key derivation, subject to implementation and security review.
- Generate a random archive key and wrap it with a password-derived key; use unique nonces per encryption operation.
- Encrypt sensitive metadata as well as file contents.
- Support password change by rewrapping the archive key rather than re-encrypting all payload data.
- Design a recovery-key workflow and document its threat model, backup instructions, and loss implications.
- Wrong-password and tamper/corruption cases must fail safely without modifying restore destinations.

## 6. Android connectivity and constraints

- Support USB through ADB and authenticated Wi-Fi transfer. MTP alone is not sufficient for robust resumable automation.
- Root is optional. Non-root access to user-selected/shared storage is a first-class supported mode.
- Android app-private data, protected system directories, and full device state are not guaranteed to be portable or restorable.
- Exclude the top-level shared-storage `Android/` directory by default in the initial policy; show exclusions clearly and allow only access Android permissions actually grant.
- Android companion app handles user consent, storage access, source selection, and reliable transfer.

## 7. Proposed architecture (validate with a technical spike)

- **Windows UI:** C# / WinUI 3.
- **Core engine:** Rust for scanning, hashing, compression, deduplication, archive I/O, encryption orchestration, and restore.
- **Android companion:** Kotlin.
- Keep UI, archive format, transport, and core processing separated behind stable interfaces.
- Use a single coordinated archive writer, bounded queues, streaming I/O, and resource-adaptive workers to prevent memory spikes and archive corruption.

## 8. Reliability and safety

- Persist checkpoints and a transaction journal for resumability.
- Finalize archives atomically where possible; incomplete archives must be detectable and recoverable or safely discarded.
- Detect insufficient destination space before and during work.
- Never delete original source files as a side effect of backup.
- Avoid duplicate uploads/writes after retry; make operations idempotent where practical.
- Verify data after restore and report per-file outcomes.

## 9. Milestones

| Milestone | Scope |
|---|---|
| M0 | Repository foundation, decisions, CI, architecture spike |
| M1 | Windows application shell and settings |
| M2 | File/folder scanner, filters, metadata capture |
| M3 | Streaming pipeline, progress, cancellation |
| M4 | Whole-file deduplication; evaluate CDC |
| M5 | Versioned ABX archive format and independent restore |
| M6 | Encryption, password handling, recovery-key design |
| M7 | Incremental snapshots, journaling, resume |
| M8 | Selective restore, verification, corruption handling |
| M9 | Android companion, ADB and authenticated Wi-Fi |
| M10 | Performance, compatibility, packaging, release |

## 10. Acceptance and test plan

- Validate byte-for-byte hash equality after restore across all four transfer directions.
- Test large datasets (target 40–80 GB where hardware permits), many small files, hidden files, duplicate files, long paths, and unusual filenames.
- Test full and incremental restore without missing dependencies.
- Test wrong password, damaged archive, interrupted finalization, and archive verification.
- Test pause/resume after process termination, reboot, USB disconnect, and Wi-Fi interruption.
- Test source-file modification during backup, destination full, permission denial, and selective restore.
- Test Android root and non-root modes separately; verify excluded paths and permission boundaries.
- Measure throughput, CPU, RAM, temporary disk usage, and deduplication effectiveness on representative data.

## 11. Non-goals and explicit limitations

- No promise of full Android app-private-data backup or universal login/session restoration.
- No full Windows/Android OS image migration in the initial product.
- No lossy compression or silent deletion of originals.
- No claim that all Android directories are accessible without root or special system privileges.

## 12. Decisions to validate before implementation

1. Confirm ABX container layout, versioning, index strategy, and recovery from interrupted writes.
2. Benchmark Rust/WinUI interop and Android transport options with a small end-to-end prototype.
3. Threat-model password recovery and key loss before freezing the encryption format.
4. Decide whether CDC is justified by measured storage savings versus CPU and restore complexity.
5. Define a compatibility matrix for Windows versions, Android versions, ADB versions, and filesystem metadata.
