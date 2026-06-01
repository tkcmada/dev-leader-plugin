# AC perspectives — 6 viewpoint checklist

When writing AC, sweep through the following viewpoints. **You do not need every viewpoint to apply**, but do not skip any viewpoint that does apply.

## TOC

- [1. Functional](#1-functional)
- [2. Behavior / Performance](#2-behavior--performance)
- [3. Error / Edge cases](#3-error--edge-cases)
- [4. Observability / Operations](#4-observability--operations)
- [5. Verification means](#5-verification-means)
- [6. Documentation (when applicable)](#6-documentation-when-applicable)

## 1. Functional
- [ ] The main use case works end-to-end
- [ ] I/O contract (type, format, range) is satisfied
- [ ] Existing functionality is preserved (no breaking changes)

## 2. Behavior / Performance
- [ ] Latency / processing time is within budget
- [ ] Memory / storage usage is within limits

## 3. Error / Edge cases
- [ ] Expected error cases behave gracefully (no crash, logged)
- [ ] Behavior is defined for empty / invalid / timed-out inputs
- [ ] Existing failure modes are not regressed

## 4. Observability / Operations
- [ ] Necessary information is logged
- [ ] Failures are surfaced to the user (GitHub comment etc.)
- [ ] Configuration changes (if any) are documented

## 5. Verification means
- [ ] Each AC item states **how to verify it**
- [ ] Verification commands are concrete (copy-pasteable)
- [ ] Expected output / state is stated

## 6. Documentation (when applicable)
- [ ] CLAUDE.md / SKILL.md / agent definitions are updated as needed
- [ ] Paths to related skills / scripts are correct
