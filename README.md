# SEAL Automation Docs (public mirror)

This is a small **public** repo that exists for exactly one reason: the three SEAL lab
automation projects' code lives in **private** repos, so a Claude session (or anyone)
without GitHub credentials for those repos can't read their `SYSTEM_OVERVIEW.md` files
directly. This repo mirrors just those overview docs — no code, no config, no secrets —
so anyone/anything can read "how does this system work?" without needing access to the
private repos.

**This is a read-only mirror, not the source of truth.** Each file below is copied from
its private repo's `SYSTEM_OVERVIEW.md` whenever that file changes. If you want to
verify something or dig into the actual code, go to the linked private repo (you'll
need read access there).

| Project | Public mirror | Private code repo |
|---|---|---|
| SEAL Onboarding Automation V1 | [`onboarding-v1.md`](onboarding-v1.md) | https://github.com/seal-scripting/Seal-Onboarding-Automation |
| Sector Account Sharing Automation | [`sector-account-sharing.md`](sector-account-sharing.md) | https://github.com/seal-scripting/Sector-Account-Sharing-Automation |
| Battlestation Bruce Diagnostic | [`battlestation-bruce.md`](battlestation-bruce.md) | https://github.com/seal-scripting/Battlestation-Bruce-Diagnostic |
