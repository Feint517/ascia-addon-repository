# ASCIA OS — Home Assistant add-on repository

Add this URL in Home Assistant under **Settings → Add-ons → Add-on Store → ⋮ → Repositories**:

```
https://github.com/Feint517/ascia-addon-repository#main
```

## This repository is generated

Every file here is produced by CI from the ASCIA source monorepo and published by **force-push**.

- **Do not edit it, and do not open pull requests against it.** Any change made here is erased by
  the next publish. Branches are snapshots, not history.
- Each folder holds only an add-on manifest (`config.yaml`). There is **no source code** here —
  Supervisor pulls each add-on's prebuilt container image from the registry named in its
  `image:` field, and those images are private.

## Branches are release channels

Home Assistant accepts a branch in the repository URL (`<url>#<branch>`), so one repository
serves every channel:

| Branch | What it is |
|---|---|
| `main` | **stable** — moves only on a deliberate release |
| `edge` | tracks every build of the source repository's `main` |

## Support

Issues and pull requests are disabled here on purpose; this repository is an artifact, not a
project. Contact the maintainer: ASCIA <dev@ascia.example>

---

© ASCIA. All rights reserved. Published for the sole purpose of add-on distribution to ASCIA
appliances; no licence is granted.
