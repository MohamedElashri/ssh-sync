# Security and Safety Risks for sync-ssh

This document lists known risks in the current design and an initial **Plan** to address each. The goal is to prevent data loss, avoid privilege or path misuse, and keep the tool predictable under failure.

## Scope and assumptions

- No software installation or root access on the remote host.
- SSH, rsync available on both ends.
- Default macOS on the local side, Linux on the remote side.
- Two-way mode uses local watch for push and remote polling for pull.



## 1) Command injection and shell pitfalls

**Issue**  
User provided values (host, paths, flags) may be interpolated into shell commands.

**Impact**  
Arbitrary command execution or unintended rsync behavior.

**Plan**  
- Remove eval usage and build argv arrays only.  
- Validate inputs: reject whitespace and shell metacharacters in host and path.  
- Quote all arguments, no string concatenation for commands.



## 2) Archive or trash directory subversion

**Issue**  
Receiver side archive path could be a symlink or hard link that escapes the tree.

**Impact**  
Backups written outside the project or into sensitive locations.

**Plan**  
- Create archive directories with mode 0700.  
- Reject if path is a symlink or not a directory.  
- Verify device and inode remain stable between checks.



## 3) Infinite sync loops and backup ping pong

**Issue**  
Archive folders or partial files get synced back and forth.

**Impact**  
Unbounded growth and constant churn.

**Plan**  
- Hard exclude .sync-ssh/, .rsync-partials/ on both directions.  
- Keep archive and partials under excluded paths.



## 4) Deletes that reappear (zombie deletes)

**Issue**  
With safe mode, deletes on one side may be reintroduced by the other.

**Impact**  
Confusing state and accidental data reappearance.

**Plan**  
- Introduce tombstones: write .sync-ssh/tombstones/<relpath>.del on delete.  
- Sync tombstones both ways and honor them with archive then remove.  
- Add TTL and prune command.



## 5) Tearing writes observed by readers

**Issue**  
Readers may see half written files while rsync is updating.

**Impact**  
Corrupted reads or build failures.

**Plan**  
- Use rsync options: --delay-updates and --partial-dir=.rsync-partials.  
- Exclude .rsync-partials from sync.



## 6) Clock skew causes missed or wrong updates

**Issue**  
mtime based heuristics fail with significant local vs remote skew.

**Impact**  
Stale files or unnecessary transfers.

**Plan**  
- Detect skew by comparing epoch seconds via ssh.  
- Offer paranoid mode using --checksum periodically or on demand.  
- Warn if skew exceeds a threshold.



## 7) Disk exhaustion by archives and partials

**Issue**  
Archive and partial directories can grow unbounded.

**Impact**  
Disk full, aborted syncs, data loss from unrelated processes.

**Plan**  
- Retention policy by size or age.  
- Add prune subcommand and configurable limits.  
- Rotate logs with size thresholds.



## 8) Case sensitivity collisions (APFS vs Linux)

**Issue**  
Different files Foo and foo collide on case insensitive filesystems.

**Impact**  
Overwrites, loops, or silent data loss.

**Plan**  
- Preflight scan for collisions using a case fold map.  
- Refuse two way mode unless user passes an explicit override.  
- Recommend case sensitive APFS sparse image.



## 9) Unsafe symlink handling

**Issue**  
Symlinks may point outside the root or to absolute paths.

**Impact**  
Unexpected content traversal or replacement.

**Plan**  
- Use rsync --safe-links by default.  
- Optionally --copy-unsafe-links if content is preferred.  
- Always exclude archive, tombstones, and partial dirs from link resolution.



## 10) Permissions and ownership drift

**Issue**  
Mode and ownership drift surprises users or breaks tools.

**Impact**  
Hidden exposure or failing builds.

**Plan**  
- Set receiver side defaults with rsync --chmod (e.g. Du=rwx,Dgo=,Fu=rw,Fgo=).  
- Avoid owner and group transfer.  
- Add a --umask style flag mapped to chmod rules.



## 11) Unicode normalization mismatch

**Issue**  
macOS uses UTF-8-MAC (NFD). Linux typically uses NFC.

**Impact**  
Flapping file names or duplicates.

**Plan**  
- Detect macOS and prefer rsync --iconv=UTF-8-MAC,UTF-8.  
- Provide --no-iconv override.



## 12) Push and pull race conditions

**Issue**  
Push and pull can touch the same file at the same time.

**Impact**  
Confusing archives, wasted bandwidth, transient conflicts.

**Plan**  
- Serialize transfers with a single in-process queue.  
- Debounce fswatch events for 300 to 800 ms.  
- Stagger pull by a fixed delay relative to push.



## 13) Artifact churn floods the channel

**Issue**  
Large generated files update frequently.

**Impact**  
Slow sync and massive archives.

**Plan**  
- Expand default excludes and honor per project .syncignore.  
- Provide --exclude-from override.  
- Document common patterns to exclude.



## 14) SSH hardening and host key trust

**Issue**  
Weak host key checks or agent exposure raise MITM risk.

**Impact**  
Credential theft or hostile server trust.

**Plan**  
- Add --strict flag that sets StrictHostKeyChecking=yes and a dedicated known_hosts file.  
- Disable agent forwarding.  
- Allow extra ssh options but document safe examples.



## 15) Ancient rsync on macOS

**Issue**  
macOS includes rsync 2.6.9 with limitations and known **Issue**s.

**Impact**  
Missing features and potential bugs.

**Plan**  
- Prefer Homebrew rsync 3.x if available.  
- Provide --rsync to set a custom rsync path.  
- Warn if only Apple rsync is found.



## 16) Path validation and dangerous roots

**Issue**  
Typos or bad roots like / or /etc.

**Impact**  
Accidental destructive syncs.

**Plan**  
- Enforce absolute paths.  
- Deny a list of forbidden prefixes on either end.  
- Print a confirmation summary on first run unless --yes is set.



## 17) Event storms from fswatch

**Issue**  
Burst events trigger overlapping runs.

**Impact**  
High CPU and redundant transfers.

**Plan**  
- Coalesce events and rate limit rsync runs.  
- Drop triggers while a run is active.



## 18) Sensitive data leakage

**Issue**  
Archives and logs may contain secrets or sensitive paths.

**Impact**  
Disclosure risk if shared.

**Plan**  
- Exclude common secret file patterns by default.  
- Support user provided redact list for logs.  
- Document best practices in SECURITY.md.



## 19) Remote path becomes unwritable mid session

**Issue**  
Permissions or quotas change mid run.

**Impact**  
Half applied syncs and drift.

**Plan**  
- Preflight create and delete a temp file on both sides.  
- On failure, downgrade to pull only, warn clearly, and keep state consistent.



## 20) Exit codes, retries, and resumability

**Issue**  
Transient network failures interrupt syncs.

**Impact**  
Partial state and user confusion.

**Plan**  
- Retry with exponential backoff for common rsync and ssh exit codes.  
- Preserve --partial-dir to resume efficiently.  
- Show status: last success, last error, retry count.



## Implementation roadmap (first wave)

- Replace eval with argv arrays and strict input validation.  
- Add rsync flags: --delay-updates, --partial-dir=.rsync-partials, --safe-links.  
- Expand excludes: .sync-ssh/, .rsync-partials/, tombstones, archives.  
- Archive hardening: perms 0700, no symlinks, device and inode checks.  
- Debounced event handling and serialized transfer queue.  
- Detect Apple rsync; prefer brew rsync 3.x; allow --rsync override.  
- Add --strict for hard host key checks and no agent forwarding.  
- Preflight checks: case collision scan, clock skew, writable probes.

## Implementation roadmap (second wave)

- Tombstones with TTL and prune subcommand.  
- Retention policy for archives and partials with size and age limits.  
- Paranoid checksum scan mode scheduled periodically.  
- Unicode iconv auto detection and override.  
- Status metrics endpoint or simple text status file for observability.


