# devcontainer-template

A reusable, network-isolated devcontainer setup for running Claude Code (or any AI coding
assistant) against a project without giving it access to anything outside that project.

## Why this exists

Built after discovering an AI coding assistant read a file it had been explicitly told not to
touch, and its contents left the machine. The response wasn't "trust the tool more carefully" —
it was "make it structurally impossible for the tool to reach anything sensitive, regardless of
what it's told or tricked into doing."

## What this actually protects against, and what it doesn't

- **Filesystem isolation is real and structural, not a permission that can be talked around.**
  The container only ever sees what's explicitly mounted into it (this project folder, plus a
  couple of named volumes for config/credentials). There is no mount to your Windows drive, your
  broader WSL home directory, or any other project. Root inside the container changes nothing
  here — privilege only matters within what's already visible, and nothing expands that from
  inside a running container. See the "how mounts work" discussion — the only way this boundary
  changes is editing `devcontainer.json` itself and then rebuilding, which is why that file's
  diffs are worth actually reading before agreeing to a rebuild.
- **Network egress is restricted by `init-firewall.sh`**, an iptables-based allowlist that denies
  outbound traffic by default and only permits specific domains (GitHub, PyPI, npm, Anthropic's
  API/auth domains, VS Code's own update infrastructure). Without this, anything running inside
  the container — including a compromised or tricked AI session — could exfiltrate data or
  credentials to any arbitrary server. Adding a new dependency from a new source requires adding
  its domain here explicitly; a silent network failure after adding a package usually means the
  firewall, not a broken install.
- **Git credentials are scoped, not broad.** Use a repo-specific SSH deploy key (generated inside
  the container, added to that one repo's GitHub deploy keys with write access), not a personal
  access token or `gh auth login`. A leaked deploy key only exposes the one repo; a leaked broad
  token exposes your whole account.
- **What this does NOT protect against:** anything mounted into the container (this project's own
  files) can still be read, modified, or deleted by anything running inside it — the isolation is
  about the boundary around the container, not about trusting everything inside it completely.
  Push to git regularly; that's your actual recovery path if something in the project folder gets
  corrupted or wiped, not the sandboxing itself. Also: passwordless `sudo` is enabled inside the
  container by default for convenience (fixing volume ownership issues, installing packages) —
  this means root-level access within the container isn't meaningfully gated. A real tradeoff,
  left as a deliberate choice rather than an oversight; revisit if this project's risk profile
  changes.

## Using this for a new project

1. Copy this repo's `.devcontainer/` folder into your new project.
2. Rename the volume identifiers in `devcontainer.json` if you want per-project isolation of
   Claude Code's login and SSH key (recommended default) — the `${devcontainerId}` variable
   already does this automatically; you don't need to touch it.
3. Open the project in VS Code, connected to WSL (not a plain Windows-local window — check for
   the "WSL: Ubuntu" badge in the status bar before doing anything else).
4. Command Palette → **Dev Containers: Reopen in Container**. First build takes a few minutes.
5. Inside the container: generate a fresh SSH deploy key
   (`ssh-keygen -t ed25519 -f ~/.ssh/<project>_key`, then `chmod 600` on the private key), add
   its public half to the new GitHub repo's deploy keys with write access, set
   `git config core.sshCommand "ssh -i ~/.ssh/<project>_key"`, add the remote, and push.
6. Log into Claude Code (`claude`) — this is a fresh container identity, so it'll need its own
   login even if you've logged in elsewhere before.
7. Update the firewall's domain list in `init-firewall.sh` if this project needs to reach
   anything beyond what's already allowed (a new package registry, a different API).

## Files

- `devcontainer.json` — container definition: base image, features (Node + Claude Code CLI),
  non-root user, persistent volumes for `.claude` and `.ssh` (so logins and keys survive a
  rebuild), and the commands that fix volume ownership and run the firewall on every start.
- `init-firewall.sh` — the network allowlist itself. Read the comments inline; the domain list
  near the bottom is the part you'll actually edit per-project.