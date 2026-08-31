# claude-box

Runs Claude Code inside a rootless podman container. Only the directories you
name are visible to it.

## Requirements

- **rootless podman**
- **Claude Code installed on the host.** The `claude` binary is a single
  self-contained file, so it is mounted into the container rather than
  installed in the image. The default location is `/opt/claude-code` (where
  the Arch `claude-code` package puts it). If yours is elsewhere, set
  `CLAUDE_BOX_CLAUDE_DIR` to the directory containing `bin/claude`.
- **Linux**, with your home directory at `/home/$USER` and not behind a
  symlink. Mounts are resolved with `realpath`, so where `/home` is itself a
  symlink (Fedora Atomic puts it at `/var/home`) the container sees the
  resolved paths, and pixi prefixes and session history keys may not match.
  The script warns in both cases but does not try to fix them.
  See the glibc note below for why the image is Arch-based.
- optional: `pixi` on PATH, and `nvidia-container-toolkit` for GPU work.

### The glibc caveat

The `claude` binary is dynamically linked, and inside the container it runs
against the *image's* glibc. Your host's glibc version does not come into it:
the binary is prebuilt by Anthropic, not compiled against your system.

So the only requirement is that the image's glibc is at least as new as the
one Anthropic built against. Today that bar is low (the current binary asks
for nothing newer than `GLIBC_2.26`, from 2017), and Arch is used as the base
because it keeps a recent glibc with plenty of headroom.

The realistic failure is a **stale image**, not a mismatched host. The image's
glibc is fixed when you build it, so a much later Claude release could want
something newer than the image has, and would fail at startup with
`version 'GLIBC_2.xx' not found` rather than any container-related error. The
fix is to rebuild:

    ./claude-box --build

`--build` passes `--pull=newer`, so this also refreshes the Arch base image.

## Setup

    ./claude-box --build

The build reads your username, UID and GID (`id -un`, `id -u`, `id -g`) and
passes them as build args, so the container user is a copy of you. This
matters; see [Why the paths must match exactly](#why-the-paths-must-match-exactly).

**The image is therefore per-user: everyone builds their own.** This costs
nothing in practice, since rootless podman keeps a separate image store per
user anyway.

Then symlink the scripts onto your PATH. The short names are what you type;
the long name keeps the directory self-explanatory later:

    ln -s /path/to/claude-box/claude-box ~/.local/bin/cbox
    ln -s /path/to/claude-box/cbox-obs   ~/.local/bin/cbox-obs

`claude-box` and `cbox` are the same script.

## Usage

`claude-box` (`cbox`) is the general tool; `cbox-obs` is a preset for
OpenBioSim.

### cbox-obs: OpenBioSim development

    cd $OBS_ROOT/sire
    cbox-obs

A small wrapper that sets four things, all specific to OpenBioSim:

- mounts the whole stack at `$OBS_ROOT` (default `~/Code/openbiosim`)
- `CLAUDE_BOX_ENV=dev`, the only OBS development environment
- `CLAUDE_BOX_MANIFEST=$OBS_ROOT/sire/pixi.toml`, so the environment always
  comes from sire
- `CLAUDE_BOX_GPU=1`, for CUDA pass-through

The manifest setting matters in packages that have their own `pixi.toml`. If
you launch from biosimspace without it, `pixi run` would use biosimspace's
environment, but OBS development needs sire's. Pinning the manifest keeps you
in the directory you started from while the environment comes from sire.

Each of these is an ordinary environment variable, so setting one yourself
overrides the preset: `CLAUDE_BOX_ENV=test cbox-obs`.

If your checkout is somewhere else, set `OBS_ROOT`, either per run or once in
your shell profile:

    OBS_ROOT=~/src/openbiosim cbox-obs
    export OBS_ROOT=~/src/openbiosim

### cbox: anything else

    cd ~/src/openmm
    cbox

With no `-m`, the current directory is mounted. If you run it from inside a
mounted directory it starts there; otherwise it starts at the first mount.
Anything after the `claude-box` options is passed on to `claude`, so
`cbox --continue` and `cbox --resume` work as usual. Put `-m`, `--env` and
`--manifest` first; parsing stops at the first option `claude-box` does not
recognise.

Mount other directories with `-m` / `--mount`, which can be repeated:

    claude-box -m ~/src/openmm
    claude-box -m ~/src/openmm -m ~/src/emle-engine

Or change the default:

    CLAUDE_BOX_MOUNTS=~/src/openmm:~/src/peptone claude-box

Directories nested inside another mount are dropped, as are duplicates, so the
mounts you name never overlap each other. (The read-only git mounts do sit on
top of them, deliberately; see
[Read-only git internals](#read-only-git-internals).)

Mounts are refused if they would defeat the point of the container: `/`, your
home directory, anything containing it (`/home`, or `~/..`), and `~/.ssh`,
`~/.gnupg`, `~/.aws`, `~/.kube` or `~/.config/gh`. The check runs after
resolving symlinks, so it cannot be sidestepped by pointing at one. That list
is not exhaustive and is not meant to be: the boundary is that nothing is
mounted unless you name it, so think before naming a directory of dotfiles.

### Launching inside a pixi environment

    cbox --env dev            # or CLAUDE_BOX_ENV=dev

Claude is started *inside* the activated environment, so every shell it opens
inherits it and there is no need to prefix commands with `pixi run -e ...`.
`pixi shell` cannot do this, because it is an interactive subshell that would
not carry over to the next command, so the entrypoint uses `pixi run` instead.

`--no-install --frozen` is passed, so pixi can never download, build or touch
the lock file: the environment must already exist on disk (it is mounted from
the host). If no pixi manifest is found at or above the working directory, and
within the mounted tree, the script tells you before starting the container.

By default pixi searches upward from the working directory, which finds a
subpackage's own manifest. To take the environment from somewhere else while
staying in the current directory, pin one with `--manifest` (or
`CLAUDE_BOX_MANIFEST`):

    cbox --env dev --manifest ~/openbiosim/sire/pixi.toml

The manifest must be inside one of the mounts, or the container cannot see
it. The script checks and tells you if not.

## What gets mounted

| Host | Mode | Why |
|---|---|---|
| `$CLAUDE_BOX_CLAUDE_DIR` (default `/opt/claude-code`) | ro | Claude itself: a self-contained native binary |
| `~/.claude`, `~/.claude.json` | rw | Auth, settings, session history |
| the directories you name | rw | Your work, each at the same path as on the host |
| `config`, `hooks`, `commondir` in each gitdir under a mount | ro | They run on the host later |
| a worktree's or submodule's `.git` file | ro | It names the gitdir, so it must not be repointed |
| each repo directory, gitdir and worktree metadata directory | rw, pinned | Bound onto themselves so they cannot be renamed aside |
| `~/.gitconfig`, `~/.config/git/config` (whichever exist) | ro | Commit identity |
| `pixi` (wherever it is on PATH) | ro | Run project environments |
| `$CLAUDE_BOX_CUDA_DIR` (default `/opt/cuda`), at `/opt/cuda` | ro | CUDA toolkit, only with `CLAUDE_BOX_GPU=1` |

Nothing else in `$HOME` is visible. In particular, **not** `~/.ssh` and
**not** `~/.gnupg`.

### Environment

`TERM` is forwarded, and the timezone is taken from the host with
`--tz=local`, since most systems set it through `/etc/localtime` rather than
`$TZ`. Setting `TZ` to a zoneinfo name such as `Europe/London` overrides that;
a POSIX-style value like `GMT0BST` is ignored, as podman would reject it.
Forward anything else, such as `HTTPS_PROXY` or Bedrock and Vertex settings,
by name:

    CLAUDE_BOX_PASSENV=HTTPS_PROXY:NO_PROXY cbox

Variables are passed by name rather than value, so nothing sensitive shows up
in the host's process list. `LANG` is deliberately not forwarded: the image
carries only `C.UTF-8`, and pointing `LANG` at a locale it does not have is
worse than leaving it alone.

`ANTHROPIC_API_KEY` is forwarded automatically when set. Worth knowing if you
keep one exported for other work: container sessions will then bill the API
rather than your subscription. Unset it for the launch if that is not what you
want.

On SELinux systems the script adds `--security-opt label=disable`, without
which rootless bind mounts are denied. Relabelling with `:Z` is not used, as
it would break access to the same files from the host. The trade-off is that
the container loses SELinux confinement; the user namespace and the mount set
are still what is doing the work.

## herdr

Sessions started with `cbox` from a herdr pane appear in the agents panel like
any other. Two things are needed, and the second is the important one:

1. The `herdr-agent-state.sh` hook reports the session id over a Unix socket,
   so the `HERDR_*` variables are passed through and `$HERDR_SOCKET_PATH` is
   mounted at its own path. `--userns=keep-id` gives the container user
   ownership of the socket, and `python3` (which the hook needs) is in the
   image.

2. herdr decides whether a pane holds an agent by the name of its
   **foreground process**, which for a container would be `podman`. Linux
   names a process after the file it was started from, so the script runs
   podman through a symlink named `claude` at `~/.cache/claude-box/claude` on
   the host. The pane then reports `process=claude` and herdr picks it up.

Without the second step, the socket report is accepted and then thrown away:
it only attaches a session id to a pane that is already known to hold an
agent.

Both only happen when `HERDR_ENV=1` and the socket exists, so outside herdr
nothing changes: podman is run directly and no symlink is created.

## Session history and memory

These carry over in both directions without any extra steps. Claude Code
stores history under `~/.claude/projects/`, in a folder named after the
working directory with slashes turned into dashes, so
`/home/alice/openbiosim/sire` becomes
`-home-alice-openbiosim-sire`.

Because mounts keep their exact host paths and `~/.claude` is mounted
read-write (the same directory on disk, not a copy), the container ends up
using the same folder names. `claude --resume` and `-c` see existing sessions,
new sessions appear in the host's history, and per-project memory under
`~/.claude/projects/<key>/memory/` works from either side. The global
`~/.claude/CLAUDE.md` comes along too; per-project `CLAUDE.md` files live in
the repos, which are mounted anyway.

This depends on the paths matching. A directory mounted at a different
container path would quietly start a fresh, empty history under a new name.

## Why the paths must match exactly

Pixi/conda environments **cannot be moved**. Absolute paths are written into
script shebangs, ELF `RUNPATH`s, `.pc` files and CMake configs. So the
container user is a copy of the host user (same name, same UID/GID, home at
`/home/$USER`) and every mount appears at its real path. For alice,
`~/openbiosim/sire/.pixi` must be
`/home/alice/openbiosim/sire/.pixi` inside the container, or the
environments break. The same rule is what keeps session history working
(above), and why `--userns=keep-id` makes files written inside the container
belong to you.

This is why the image is built per-user, using the `USERNAME`/`UID`/`GID`
build args the script fills in from `id`. A shared prebuilt image would have
one person's home path baked in and break everyone else's environments.

It is also why the environments are mounted rather than copied into the
image: `sire/.pixi` alone is 21 GB, and a copy would go stale on the next
`pixi install`.

## Committing and signing

The mount is the same working tree, not a copy, and `--userns=keep-id` means
Claude's edits land on the host owned by you. So **commit from a normal host
terminal**. `git commit -S` signs there as usual, because `~/.gnupg` is on the
host where it always was.

GPG signing cannot work *inside* the container, by design. If Claude needs to
commit in there, it must be unsigned:

    git -c commit.gpgsign=false commit ...

Avoid running git on the host and in the container at the same time, or you
will occasionally hit an `index.lock` error. It is harmless; just retry.

### Read-only git internals

Git executes what it finds in a repo's `.git`: the hooks, and config keys such
as `core.hooksPath`, `core.pager`, `core.fsmonitor` and `core.sshCommand`.
Those run on the *host*, with your keys, the next time you commit there. Since
the advice is to commit from the host, that would undo much of the point of
the container.

So every `.git/config` and `.git/hooks` under a mount is mounted read-only on
top of the repo. The script finds them up to four directories deep, which
covers a mount holding a whole stack of repos.

Read-only binds alone would not be enough, because the directory holding them
can be renamed aside and replaced with a writable copy:

    mv .git .git_old && cp -a .git_old .git

So `.git`, the repo directory and every directory between it and the mount
root are also bind-mounted onto themselves. A mountpoint cannot be renamed, so
those moves fail with `Device or resource busy`. If `.git/hooks` does not
exist yet, the script creates it on the host first, since an empty path cannot
be covered.

There is one more redirect to close. Git reads `$GIT_DIR/commondir`, and takes
config and hooks from whatever directory it names, which skips the read-only
mounts entirely. `.git` itself has to stay writable for `index.lock` and the
rest, so the file could otherwise just be created. The script therefore writes
`.` into `.git/commondir` when it is absent, and mounts it read-only. `.`
means "look in `.git`", which is what git does anyway, so nothing changes;
`git rev-parse --git-common-dir`, ordinary commits and `git worktree add` all
behave as before.

Worktrees and submodules get the same treatment, because each is another way
in:

- `.git/worktrees/<name>/commondir` redirects a linked worktree, so it is
  read-only, and `.git/worktrees` and each `<name>` directory are pinned so
  they cannot be replaced wholesale. The `gitdir` back-pointer beside it is
  read-only too: rewriting it runs nothing, but it marks the worktree
  prunable, and the next `git worktree prune` or `git gc` would then throw
  its metadata away.
- A worktree or submodule checkout has `.git` as a *file* naming its real
  gitdir. Those are mounted read-only, so the container cannot repoint one at
  a gitdir it controls.
- Each submodule has a full gitdir under `.git/modules/<name>`, with its own
  config and hooks, so each one is protected exactly like a repo. Nested
  submodules are found by their config file and handled too.

The cost is a small permanent file in every protected repo. It is inside
`.git`, so nothing tracks it, but tools that parse `.git` themselves rather
than shelling out to git have not been tested against it.

Reading is unaffected, which is what the container is for: `git status`,
`git diff`, `git log` and `git show` all work, and so does the index, so
committing works too, including `git -c commit.gpgsign=false commit`. What
fails inside is writing config: `git config`, `git remote add`, and setting a
branch upstream. `git worktree move` and `git worktree repair` fail too, since
both rewrite the pinned back-pointers. Do those on the host, or set
`CLAUDE_BOX_UNSAFE_GIT=1` to turn the protection off.

Two gaps worth knowing. A repo more than four directories below a mount is not
found, and neither is a submodule gitdir nested more than six below
`.git/modules`. And a repo created *inside* the container, by `git init` or
`git clone`, has a fully writable `.git`, because the mounts are fixed when
the container starts.

A worktree whose checkout lives outside every mount is not reachable from the
container at all, so it needs nothing; one inside a mount is covered by the
read-only `.git` file and the pinned metadata above.

## GPU

Run with:

    CLAUDE_BOX_GPU=1 claude-box

All GPUs are passed through. To limit which one CUDA uses inside, set
`CUDA_VISIBLE_DEVICES` on the host; it is forwarded into the container when
`CLAUDE_BOX_GPU=1`.

CDI provides only the driver. The CUDA toolkit (`nvcc` and friends) is mounted
read-only from the host at `/opt/cuda` inside the container, and the image
puts `/opt/cuda/bin` on `PATH`, so `nvcc` works without any per-shell setup.
The host location defaults to `/opt/cuda` (Arch's `cuda` package); point
`CLAUDE_BOX_CUDA_DIR` elsewhere if yours differs, e.g.
`CLAUDE_BOX_CUDA_DIR=/usr/local/cuda-12.8`. It is skipped if the directory does
not exist.

The one-time host setup, for reference or for another machine:

    sudo pacman -S nvidia-container-toolkit
    sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

To check the GPU is visible without involving Claude:

    podman run --rm --device nvidia.com/gpu=all \
        --entrypoint nvidia-smi "${CLAUDE_BOX_IMAGE:-claude-box:latest}"

`nvidia-smi` is not in the image; CDI provides it along with the driver
libraries and devices.

### The CUDA pixi environments need it

sire's `dev` environment requires CUDA for OpenMM. Without the GPU flag there
is no `/dev/nvidia*`, pixi sets no `__cuda` virtual package, and activation
fails with *"environment 'dev' does not support 'linux-64' on this machine"*.
Run with `CLAUDE_BOX_GPU=1` and it activates normally.

Do **not** work around this with `CONDA_OVERRIDE_CUDA=12`. It only fools the
solver: the environment installs, but nothing in it can run, because there is
still no driver or device.

The CDI spec must match the installed driver, or the container stops seeing
the GPU. On Arch the `nvidia-container-toolkit` package ships a pacman hook
that updates it after each `nvidia-utils` upgrade, so nothing to do there. On
other distros, or if the GPU disappears after a driver update, regenerate it:

    sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

## Security: what this does and does not give you

The container does **not** reduce what Claude is allowed to do. It runs as
your UID, `--userns=keep-id` maps that to you, and writes to mounted repos are
exactly the same as without a container. What changes is *reach*: only what is
mounted can be seen.

**Protected:** everything else in `$HOME`, in particular `~/.ssh` and
`~/.gnupg`, meaning GitHub push access and your commit-signing identity. That
is the main win.

**Not protected:**

- Every mounted repo. A bad `rm -rf` or `git reset --hard` hits them exactly
  as it would on the host. Git remotes are the recovery, not this.
- **What the container writes into a repo can still run on the host.** Git's
  own entry points are closed off for repos that exist when the container
  starts: `.git/config` and `.git/hooks` are read-only, and the directories
  around them are pinned so they cannot be swapped out
  (see [Read-only git internals](#read-only-git-internals), which lists what
  that does not cover). Other tools that execute repo-local configuration are
  not covered, because those are the project's own files and Claude is meant
  to edit them: `.envrc` under direnv, `pixi.toml` tasks, `Makefile`s,
  `package.json` scripts, `conftest.py`. Review changes to those as you would
  a pull request from a stranger.
- `~/.claude` and `~/.claude.json` are mounted read-write. Besides OAuth
  credentials, they hold things the *host* Claude runs: hooks in
  `settings.json`, MCP server commands in `.claude.json`, and custom commands,
  skills and plugins. A session inside the container can write a hook that
  runs on the host the next time you start Claude there. If that worries you,
  add a read-only mount of `~/.claude/settings.json` to `ARGS` in the script,
  which stops that at the cost of Claude being unable to change its own
  settings inside.
- **The network is wide open**, since Claude needs the API. The container
  shrinks what can be read; it does not stop data leaving. So against prompt
  injection, what you are covered on is your SSH and GPG keys specifically.
  Anything that *is* mounted can be read and sent out: repo contents, a `.env`
  file in a mounted project, and the Claude credentials in `~/.claude`.
- Rootless podman's user namespace is a good boundary, but not as strong as a
  VM.

### Compared with Claude Code's built-in sandbox

`/sandbox` can cover much of this without an image, CDI spec or wrapper. It is
not only a write/network boundary: `sandbox.filesystem.denyRead` /`allowRead`
and `sandbox.credentials.files` restrict reads too, and the docs give this
example:

    "sandbox": { "credentials": { "files": [
        { "path": "~/.ssh", "mode": "deny" },
        { "path": "~/.aws/credentials", "mode": "deny" } ] } }

or, more broadly, `"denyRead": ["~/"]` with `"allowRead": ["."]`.

Two differences keep the container worthwhile:

- **Default-deny vs. default-allow.** "There is no built-in credential deny
  list, so only the files and variables you list are restricted." The sandbox
  protects what you remember to list; the container protects everything you
  did not mount, with no list to maintain.
- **Scope.** "The setting affects sandboxed Bash commands only", so the Read
  tool needs its own separate deny rules to match.

They complement each other, and neither does the other's job:

- the container gives default-deny reads and no unsandboxed escape hatch
- the sandbox gives **control over outgoing network traffic**, which this
  container has none of

So the image installs `bubblewrap` and `socat`, the two packages the sandbox
needs on Linux, and you can set `sandbox.enabled: true` in
`~/.claude/settings.json`. The two then work together inside the container:
the sandbox limits writes and network access, while the container limits what
exists at all.

Running the sandbox inside a container needs one extra setting. An
unprivileged container cannot mount a fresh `/proc`, so bubblewrap fails with
`Can't mount proc on /newroot/proc: Operation not permitted`. Set
`sandbox.enableWeakerNestedSandbox: true` and the inner sandbox bind-mounts
the container's existing `/proc` instead. The weakening is that sandboxed
commands can see process information a fresh `/proc` would hide, which is an
acceptable trade here because the container is already the outer boundary.

## Shared-state warning

`~/.claude` and `~/.claude.json` are mounted read-write and shared with the
host. Running host Claude and container Claude at the same time means they
overwrite each other's changes. Avoid this especially in the same project
directory, where they share session history.

To keep the two from fighting over `~/.claude.json`:

    CLAUDE_BOX_NO_CLAUDE_JSON=1 ANTHROPIC_API_KEY=sk-... cbox

That swaps the mount for a container-only copy at
`~/.cache/claude-box/claude.json`, which persists between runs so you are not
sent through onboarding and folder-trust prompts on every launch.

It does **not** separate credentials. On Linux the OAuth tokens live in
`~/.claude/.credentials.json`, and `~/.claude` is still mounted read-write, so
both sides share it. `~/.claude.json` holds account metadata, project trust
and MCP configuration. Session history is shared too, so it still collides in
a shared project directory.

## License

MIT. See [LICENSE](LICENSE).
