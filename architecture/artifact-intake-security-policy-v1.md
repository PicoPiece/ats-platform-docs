# Artifact Intake Security Policy v1

## Purpose

Customer artifacts are untrusted input. Upload authorization and a matching
SHA-256 establish transport integrity; they do not make archive contents safe.

This policy applies before an artifact leaves quarantine or becomes eligible
for a `ValidationRun`.

## Default behavior

- Accept only media types enabled for the project/platform profile.
- Validate declared size and SHA-256 before inspection.
- Store uploads in a tenant-scoped quarantine namespace.
- Never execute uploaded files on the control plane.
- Never mount customer filesystems on the Jenkins master.
- Treat filename extensions as hints, not trusted media-type evidence.
- Reject rather than partially accept malformed content.

## Archive policy

Archives such as `.tar`, `.tar.gz`, `.tar.bz2`, `.tar.xz`, and `.zip` require an
explicit archive intake profile.

The profile defines:

- maximum compressed size;
- maximum total extracted size;
- maximum file count;
- maximum single-member size;
- maximum directory depth;
- allowed member types;
- explicit expected member names or patterns;
- whether nested archives are forbidden or separately bounded.

Initial values are project policy, not customer-controlled manifest values.

## Required archive checks

Before extraction:

1. inspect the complete member list;
2. reject absolute paths;
3. normalize every path and reject `..` traversal;
4. reject Windows drive/UNC paths where applicable;
5. reject NUL/control characters and duplicate normalized paths;
6. reject members outside the explicit expected-member policy;
7. reject device nodes, FIFOs, sockets, and other special files;
8. reject hard links unless explicitly required and proven contained;
9. reject symlinks that are absolute or can escape the extraction root;
10. calculate projected extracted size and file count;
11. account for sparse files and metadata that can expand storage usage;
12. reject unsupported compression methods and encrypted archives.

During extraction:

- use a dedicated unprivileged process/container;
- extract into a new empty directory on a quota-limited filesystem;
- do not follow symlinks;
- use no shell interpolation;
- enforce CPU, memory, wall-time, inode, and disk quotas;
- stop and delete the quarantine extraction on any limit violation.

After extraction:

- verify actual file count and total allocated size;
- verify each required member;
- calculate member SHA-256 values when members become artifacts;
- remove execute permissions unless explicitly required for storage only;
- record inspection tool/version and decision;
- do not execute any member.

## Decompression-bomb protection

Inspection must enforce both:

- absolute extracted-size/file-count limits;
- compressed-to-extracted expansion ratio limit.

A highly compressible legitimate image may need a platform-specific exception.
The exception is explicit, bounded, reviewed, and recorded; it never disables
the absolute quota.

## Expected-member policy

Prefer exact role mapping:

```yaml
archive_policy:
  expected_members:
    - path: rootfs.wic
      role: disk_image
      required: true
    - path: rootfs.wic.bmap
      role: bmap
      required: false
  allow_additional_members: false
```

Do not unpack an arbitrary archive merely to search recursively for something
that resembles a firmware image.

## Disk images

Disk images are not executed or mounted on the control plane by default.

Allowed operations in quarantine:

- media-type/format inspection through hardened tools;
- size and partition-table metadata inspection;
- checksum calculation;
- read-only scanning in a dedicated sandbox when explicitly enabled.

Provisioning transfers the accepted artifact to a fenced station Provisioner.
The customer image executes only on the DUT network and declared hardware.

## Intake result

Each artifact records:

```yaml
quarantine_status: accepted
inspection:
  policy_id: artifact-intake-v1
  media_type_detected: application/x-tar
  compressed_size: 104857600
  extracted_size: 734003200
  file_count: 2
  expected_members_valid: true
  inspected_at: "2026-10-01T10:00:00Z"
  decision: accepted
```

Rejected artifacts retain only metadata/evidence required by policy and are
deleted according to rejection retention.

## Audit events

Record:

- upload started/completed;
- size/hash mismatch;
- media-type mismatch;
- archive policy violation;
- malware/content inspection result when enabled;
- quarantine acceptance/rejection;
- deletion/retention action.

Audit logs must not include artifact contents or credentials.

## M2 acceptance gate

External artifact intake is not ready until tests cover:

- absolute-path member;
- `../` traversal;
- escaping symlink;
- hard-link escape;
- special device member;
- duplicate normalized path;
- excess extracted size;
- excess file count;
- nested/decompression bomb;
- unexpected member;
- hash and declared-size mismatch.
