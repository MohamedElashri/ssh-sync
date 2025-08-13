# sync-ssh

`sync-ssh` is a robust, rsync-based synchronization tool for keeping a local directory and a remote directory in sync over SSH.  
It works entirely from your local machine — **no installation or root access is required on the remote host**.  
It supports:
- Host selection from your SSH config (with `Include` support)
- One-shot **push** (local → remote) or **pull** (remote → local)
- Continuous **watch mode** for push, or polling for pull
- Automatic remote path creation
- Flexible ignores via built-in patterns, `.syncignore`, or rsync filter files
- Works with SSH agents, ProxyJump, and complex CERN-like multi-hop configs



## Features

- **Zero remote dependencies**: Requires only `ssh` and `mkdir -p` on the server.
- **SSH config aware**: Reads `~/.ssh/config` and `Include`d configs, lists concrete hosts for selection.
- **Flexible sync directions**:
  - **Push**: local changes go to the remote.
  - **Pull**: remote changes fetched to local.
- **Watch mode**: Instant push via `fswatch` or periodic pull with `--interval`.
- **Safe by default**: Requires absolute remote paths, supports dry-run mode, can disable deletes.
- **Network-friendly**: Bandwidth limiting and resumable transfers (`--partial`).



## Installation

### 1. Install script
```bash
mkdir -p ~/.local/bin
curl -L -o ~/.local/bin/sync-ssh \
  https://raw.githubusercontent.com/MohamedElashri/ssh-sync/main/sync-ssh
chmod +x ~/.local/bin/sync-ssh
````

Make sure `~/.local/bin` is in your `PATH`.

### 2. Install dependencies

* **Required**:

  * `ssh` (macOS built-in)
  * `rsync` (macOS built-in)
* **For watch mode**:

  * `fswatch` (macOS)

    ```bash
    brew install fswatch
    ```
* **Optional**:

  * [`fzf`](https://github.com/junegunn/fzf) for fuzzy host selection

    ```bash
    brew install fzf
    ```



## Usage

```bash
sync-ssh [options] --remote <host>:<path> --local <path>
```

### Options

| Option               | Description                                                                                              |                                                           |
| -------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `--host <ssh-host>`  | SSH host alias from your `~/.ssh/config`. Overrides host in `--remote`.                                  |                                                           |
| `--remote <spec>`    | Remote spec in `host:/abs/path` form, or just `/abs/path` if `--host` is set. Path **must** be absolute. |                                                           |
| `--local <path>`     | Local directory path (created if missing).                                                               |                                                           |
| \`--direction \<push | pull>\`                                                                                                  | Sync direction. Default: `push`.                          |
| `--watch`            | Enable watch mode (push: fswatch local; pull: periodic polling).                                         |                                                           |
| `--interval <sec>`   | Poll interval in seconds for pull watch mode. Default: 10s.                                              |                                                           |
| `--filter <file>`    | Use rsync filter file instead of built-in excludes.                                                      |                                                           |
| `--no-delete`        | Do not delete extraneous files on target.                                                                |                                                           |
| `--bwlimit <KBPS>`   | Limit bandwidth usage (in KB/s). Example: `--bwlimit 20000`.                                             |                                                           |
| \`--init \<pull      | push>\`                                                                                                  | Perform one-time initial sync before watch or single run. |
| `--ssh-opt "<opt>"`  | Extra SSH options (repeatable). Example: `--ssh-opt "-J lxplus,lbgw"`.                                   |                                                           |
| `-v`                 | Verbose rsync output.                                                                                    |                                                           |
| `--dry-run`          | Show what would change without applying.                                                                 |                                                           |
| `--list-hosts`       | List concrete hosts from SSH config and exit.                                                            |                                                           |
| `-h`, `--help`       | Show help.                                                                                               |                                                           |

## Examples

### 1. Interactive host pick, initial pull, then live push

```bash
sync-ssh \
  --remote /home/melashri/projects/code \
  --local ~/work/code \
  --direction push \
  --watch \
  --init pull
```

* Picks SSH host from your config interactively
* Pulls existing remote code to local (`--init pull`)
* Watches for local changes and pushes them to the server

---

### 2. Explicit host, one-shot push

```bash
sync-ssh \
  --host sleepy-earth \
  --remote /home/melashri/projects/code \
  --local ~/work/code \
  --direction push
```

Pushes the local directory to the remote once and exits.



### 3. Mirror from remote every 15 seconds

```bash
sync-ssh \
  --host sleepy-earth \
  --remote /srv/app \
  --local ~/mirror/app \
  --direction pull \
  --watch \
  --interval 15
```

Keeps local in sync with the remote by polling every 15 seconds.


### 4. Multi-hop CERN-style sync

```bash
sync-ssh \
  --host gpu-node \
  --remote /data/projects/allen \
  --local ~/lhcb/allen \
  --direction push \
  --ssh-opt "-J lxplus,lbgw"
```

Uses `ProxyJump` via `--ssh-opt` to go through multiple CERN gateways.



### 5. Custom ignore list

Create a `.syncignore` file in your local path:

```
*.root
*.log
tmp/
```

This is merged with the built-in ignores automatically.



## Tips

* **Host discovery**: `--list-hosts` shows all non-wildcard hosts from your SSH config, including files referenced via `Include`.
* **Case-sensitive files**: If your repo has paths that differ only by case, create a case-sensitive APFS sparse image and sync inside it:

  ```bash
  hdiutil create -type SPARSE -fs 'Case-sensitive APFS' -size 30g -volname devcase ~/devcase.sparseimage
  hdiutil attach ~/devcase.sparseimage
  ```
* **Safety**: Use `--dry-run` to preview changes before syncing.
* **Performance**: Exclude large generated files or build directories to speed up sync.


