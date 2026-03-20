# VGC vcpkg Fork

This is vlingoco's private fork of [microsoft/vcpkg](https://github.com/microsoft/vcpkg).
It provides all official vcpkg ports plus our own private `vgc-*` ports that the team
can consume just like any other vcpkg package.

---

## Why We Fork Instead of Using a Submodule

Git submodules are painful to maintain across a team. Forking gives us:

- A single repo the team clones — no submodule init/update steps
- Our own ports live alongside official ports in `ports/`
- Upstream Microsoft updates can be pulled in at any time via a second remote
- A stable, pinned baseline the whole team shares

---

## How This Fork Was Set Up

### 1. Fork on GitHub

Forked `microsoft/vcpkg` to the vlingoco organization via the GitHub UI:
`https://github.com/vlingoco/vcpkg`

> **Important:** Fork to the **organization** account, not a personal account.
> On the fork creation page, change the **Owner** dropdown to `vlingoco`.

### 2. Clone the Fork
```
git clone https://github.com/vlingoco/vcpkg C:/vcpkg
cd C:/vcpkg
```

### 3. Rename `master` to `main`

GitHub's UI rename can silently fail on forks, so do it via git:
```
git checkout master
git checkout -b main
git push origin main
```
Then in GitHub **Settings → Branches**, set `main` as the default branch, and delete `master`:
```
git push origin --delete master
git branch -d master
git remote set-head origin -a
```

### 4. Add Microsoft's Repo as Upstream
```
git remote add upstream https://github.com/microsoft/vcpkg
git fetch upstream --tags
```
You now have two remotes:
```
origin    https://github.com/vlingoco/vcpkg    ← our fork
upstream  https://github.com/microsoft/vcpkg   ← Microsoft's source
```

### 5. Pin to a Stable Release Tag

vcpkg uses date-based release tags (e.g. `2026.03.18`). List available tags:
```
git tag --sort=-version:refname | Select-Object -First 20
```
Reset `main` to the chosen stable tag and force push:
```
git reset --hard 2026.03.18
git push origin main --force
```

### 6. Bootstrap the vcpkg Tool
```
bootstrap-vcpkg.bat    # Windows
./bootstrap-vcpkg.sh   # Linux / Mac
```
This compiles the vcpkg binary from source. Must be re-run after pulling upstream updates.

---

## Adding VGC Private Ports

Add your custom ports to `ports/` using the `vgc-` prefix to avoid collisions with
upstream ports now and in the future:

```
ports/
└── vgc-my-lib/
    ├── vcpkg.json       ← name, version, dependencies (e.g. protobuf)
    └── portfile.cmake   ← how to fetch and build the library
```

After adding or updating a port, regenerate the version files:
```
vcpkg x-add-version --all --overwrite-version
```

Commit and push to `origin`. The commit SHA becomes the `baseline` that consuming
projects pin to in their `vcpkg.json`.

---

## Consuming This Registry in a Project

In any project's `vcpkg.json`:
```json
{
  "name": "my-project",
  "version": "1.0.0",
  "registries": [
    {
      "kind": "git",
      "repository": "https://github.com/vlingoco/vcpkg",
      "baseline": "<commit-sha>",
      "packages": ["vgc-my-lib"]
    }
  ],
  "dependencies": [
    "protobuf",
    "vgc-my-lib"
  ]
}
```
vcpkg resolves official packages (e.g. `protobuf`) from the built-in registry and
`vgc-*` packages from this fork, handling the dependency chain automatically.

---

## Pulling Upstream Updates from Microsoft

We have two options depending on how much risk we want to take on.

---

### Option A: Pull a Stable Release Tag (Recommended)

Microsoft publishes date-based release tags (e.g. `2026.03.18`) that represent
a tested, stable snapshot of the repo. This is the safest way to update.

**1. Fetch all upstream tags**
```
git fetch upstream --tags
```

**2. List recent tags to pick a target**
```
git tag --sort=-version:refname | Select-Object -First 20
```

**3. Check the release notes on GitHub before merging**

Go to `https://github.com/microsoft/vcpkg/releases` and review what changed.
Look for breaking changes to ports you depend on.

**4. Merge the chosen tag into `main`**
```
git checkout main
git merge 2026.06.01        ← replace with the chosen stable tag
```

**5. Resolve any conflicts**

Conflicts are rare if you've followed the `vgc-` prefix rule. If they occur
they will only be in files you've added — Microsoft's existing ports are untouched.

**6. Push and rebuild**
```
git push origin
bootstrap-vcpkg.bat
```

**7. Notify the team**

Team members pull the update and re-run bootstrap:
```
git pull
bootstrap-vcpkg.bat
```

---

### Option B: Pull Directly from Microsoft's `master`

Only do this if you need a specific fix or port update that hasn't made it into
a stable release tag yet. `master` moves daily and is not guaranteed to be stable.

```
git fetch upstream
git merge upstream/master
git push origin
bootstrap-vcpkg.bat
```

> **Warning:** Merging `master` directly means you're taking on whatever state
> it's in at that moment. Prefer waiting for the next stable tag unless you
> have a specific reason to pull from `master`.

---

### Golden Rules for Conflict-Free Merges

- **Never modify Microsoft's existing ports** — only add new `vgc-*` ports
- **Always use the `vgc-` prefix** on your ports to avoid name collisions with future upstream additions
- **Pull upstream on a regular cadence** — don't let it drift for months, large gaps make merges harder
- **Always use Option A** unless there is a specific urgent reason to use Option B

---

## Team Setup (New Developer)

```
git clone https://github.com/vlingoco/vcpkg C:/vcpkg
cd C:/vcpkg
bootstrap-vcpkg.bat
```
Set a permanent environment variable:
```
VCPKG_ROOT=C:/vcpkg
```
That's it — no submodules, no extra steps.
