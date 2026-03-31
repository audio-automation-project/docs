# Repository Deletion Gate Checklist

Use one completed checklist per repository before any hard delete from the `audio-automation-project` organization.

## Applicable Repositories (post hold period)

Candidates archived on 2026-03-31; earliest delete-eligible date: **2026-04-30** (subject to full gate passage).

| Repository | Archived | Hold ends (min) |
|------------|----------|-----------------|
| audio-automation-project | Yes | 2026-04-30 |
| audio-preparation-pipeline | Yes | 2026-04-30 |
| Automation | Yes | 2026-04-30 |
| boosty-automation | Yes | 2026-04-30 |
| SV2TTS | Yes | 2026-04-30 |

## Preconditions (all required)

- [ ] Backup artifact exists (`git bundle` or `--mirror` clone) with recorded SHA-256 checksum
- [ ] Backup stored in immutable/WORM-capable storage with access-control evidence
- [ ] Restore drill completed successfully (log link attached)
- [ ] Dependency closure report complete (code, workflows, IaC, webhooks, secrets, external jobs)
- [ ] No open blockers; migration tasks closed with owner and tracking ID

## Approvals (all required)

- [ ] Technical owner: name, date (UTC)
- [ ] Org admin approver: name, date (UTC)
- [ ] Optional: `deletion-gate.yml` workflow run ID from `audio-platform-cicd` (when implemented)

## Execution (two-person rule)

- [ ] Primary executor identity recorded
- [ ] Independent verifier identity recorded (different user)
- [ ] Pre-delete verification: repo name, org, `archived=true` matches approved target
- [ ] Deletion executed at UTC timestamp: _______________
- [ ] Post-delete evidence link (audit log or confirmation): _______________

## Post-delete

- [ ] Immutable backup retained for at least 90 days
- [ ] `migration-status.md` and this checklist updated with final status
