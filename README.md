# sync-ssh

`sync-ssh` is a robust, rsync-based synchronization tool for keeping a local directory and a remote directory in sync over SSH.  
It requires **no installation or root access on the remote host** — only `ssh` and `rsync` must be available remotely.  

It supports:
- Host selection from your SSH config (with `Include` support)
- One-shot **push** (local → remote) or **pull** (remote → local)
- Continuous **watch** mode for push, or periodic polling for pull
- **Two-way** mode in a single process, with safety features to avoid data loss



## Features

- **No remote installs**: works with just `ssh` and `rsync`.
- **SSH config aware**: reads `~/.ssh/config` and included files.
- **Multiple modes**:
  - **Push**: local changes → remote
  - **Pull**: remote changes → local
  - **Two-way**: local changes are pushed instantly, remote changes are polled and pulled
- **Safety defaults**:
  - No hard deletes by default (`--hard-delete` to enable)
  - All overwritten or deleted files on the receiving side are archived with timestamps
- **Custom ignores**:
  - Built-in patterns (`.git/`, `build*/`, `.venv/`, etc.)
  - `.syncignore` file in the local directory
  - Custom rsync filter file via `--filter`



## Installation

```bash
mkdir -p ~/.local/bin
curl -L -o ~/.local/bin/sync-ssh \
  https://raw.githubusercontent.com/MohamedElashri/sync-ssh/main/sync-ssh
chmod +x ~/.local/bin/sync-ssh
````

Dependencies:

* **Required**: `ssh`, `rsync`
* **For watch mode**: `fswatch`

  ```bash
  brew install fswatch
  ```
* **Optional**: [`fzf`](https://github.com/junegunn/fzf) for fuzzy host selection

  ```bash
  brew install fzf
  ```


## Usage

```bash
sync-ssh [options] --remote <host>:/absolute/remote/path --local /absolute/local/path
```

### Common options

| Option                       | Description                                                                  |
| ---------------------------- | ---------------------------------------------------------------------------- |
| `--host <ssh-host>`          | Host alias from SSH config (overrides the host part of `--remote`).          |
| `--remote <spec>`            | `host:/abs/path` or `/abs/path` when `--host` is set. Path must be absolute. |
| `--local <path>`             | Absolute local path (created if missing).                                    |
| `--direction <push \| pull>` | One-way sync direction (default: push).                                      |
| `--watch`                    | Continuous mode. For push: watch local; for pull: poll remote.               |
| `--interval <sec>`           | Poll interval for pull watch mode (default: 10).                             |
| `--filter <file>`            | Use an rsync filter file instead of built-in excludes.                       |
| `--hard-delete`              | Allow real deletions on the receiving side (disabled by default).            |
| `--bwlimit <KBPS>`           | Limit bandwidth usage (KB/s).                                                |
| `--init <pull \| push>`      | One-time initial sync before starting continuous mode.                       |
| `--ssh-opt "<opt>"`          | Extra SSH options, e.g. `--ssh-opt "-J jump1,jump2"`.                        |
| `--dry-run`                  | Show changes without applying.                                               |
| `-v`                         | Verbose rsync output.                                                        |
| `--list-hosts`               | List concrete SSH hosts from config and exit.                                |

---

### Two-way options

| Option                  | Description                                                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `--two-way`             | Run push watcher and pull poller in a single process.                                                                          |
| `--pull-interval <sec>` | Poll interval for remote changes in two-way mode (default: 10).                                                                |
| `--archive-dir <path>`  | Receiver-side archive dir for overwritten/deleted files. Defaults to `.sync-ssh/archive/<timestamp>` under the receiving root. |


## Examples

### One-time push

```bash
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --direction push
```

### Continuous push only

```bash
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --direction push \
  --watch
```

### Continuous pull only

```bash
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --direction pull \
  --watch \
  --interval 15
```

### Safe two-way sync in one process

```bash
# Initial pull to seed local copy
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --direction pull \
  --init pull

# Start two-way sync
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --two-way \
  --pull-interval 10
```

### Two-way with hard deletes enabled

```bash
sync-ssh \
  --host myserver \
  --remote /var/www/project \
  --local ~/projects/project \
  --two-way \
  --pull-interval 10 \
  --hard-delete
```



## Ignore files

You can create a `.syncignore` file in the local root to define additional ignores. Example:

```
*.log
tmp/
dist/
```

This file uses rsync’s filter syntax.



## Safety and conflict handling

* **No hard deletes by default** — files removed from one side are moved to the archive dir on the receiving side.
* **Backups on overwrite** — when a file is overwritten, the old version is saved in the archive dir with a timestamp.
* **Conflict resolution** — last writer wins, but backups mean you can recover any overwritten version.
