# RNOH-document — agent guide

Plain markdown documentation repo — no build system, no tests, no CI.

## Structure

```
README.md          — one-line description
学习记录.md        — blank, for future study notes
环境搭建.md        — the primary asset: 8 verified RNOH integration troubleshooting guides
```

## Conventions

- All content is in **Chinese** (Simplified).
- `环境搭建.md` is the only file with substantive content. When asked about RNOH setup, Metro, bundle sync, etc., read it first.
- Each problem entry in `环境搭建.md` has a consistent structure: background → root cause → fix → file references.
- Add new entries at the end of `环境搭建.md` following the same `## 问题N：标题` format.
- No frameworks, no dependencies, no scripts. Edits are markdown-only.
