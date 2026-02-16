## Usage
```sh
$ git wt                            # List all worktrees
$ git wt --json                     # List all worktrees in JSON format
$ git wt <branch|worktree|path>     # Switch to worktree (create worktree/branch if needed)
$ git wt -d <branch|worktree|path>  # Delete worktree and branch (safe)
$ git wt -D <branch|worktree|path>  # Force delete worktree and branch
```

The target can be specified as:

branch: a git branch name — eg. git wt feature-branch
worktree: a directory name relative to wt.basedir (default .wt) — eg. git wt some-worktree-folder-name
path: a filesystem path (absolute or relative to the current working directory) to an existing worktree — eg. git wt ../sibling, git wt /absolute/path
When deleting, the same target types apply: git wt -d feature-branch, git wt -d ., git wt -d ../sibling

## Note

Bare repositories are not currently supported. Bare repository support is planned: see #130 for details.

## Note

The default branch (e.g., main, master) is protected from accidental deletion.

If the default branch has a worktree, the worktree is deleted but the branch is preserved.
If the default branch has no worktree, deletion is blocked entirely.
Use --allow-delete-default to override this protection and delete the branch.

## Configuration

Configuration is done via `git config`. All config options can be overridden with flags for a single invocation.

#### `wt.basedir` / `--basedir`

Worktree base directory.

``` console
$ git config wt.basedir "../{gitroot}-worktrees"
# or override for a single invocation
$ git wt --basedir="/tmp/worktrees" feature-branch
```

Supported template variables:
- `{gitroot}`: repository root directory name

Default: `.wt`

> [!NOTE]
> When placing worktrees inside the repository (e.g., `.wt`), be aware of these limitations:
> - **Configuration files loaded multiple times**: Tools that traverse parent directories (e.g., Claude Code reading `CLAUDE.md`) may load configuration files from both the worktree and the main repository.
> - **Linters/formatters scanning worktree directories**: Some tools may scan worktree directories, causing slower performance or unexpected results.
>
> Placing worktrees inside the `.git` directory (e.g., `.git/wt`) resolves these issues, as most tools ignore `.git`.

#### `wt.copyignored` / `--copyignored`

Copy files ignored by `.gitignore` (e.g., `.env`) to new worktrees.

``` console
$ git config wt.copyignored true
# or override for a single invocation
$ git wt --copyignored feature-branch
$ git wt --copyignored=false feature-branch  # explicitly disable
```

Default: `false`

#### `wt.copyuntracked` / `--copyuntracked`

Copy untracked files (not yet added to git) to new worktrees.

``` console
$ git config wt.copyuntracked true
# or override for a single invocation
$ git wt --copyuntracked feature-branch
$ git wt --copyuntracked=false feature-branch  # explicitly disable
```

Default: `false`

#### `wt.copymodified` / `--copymodified`

Copy modified files (tracked but with uncommitted changes) to new worktrees.

``` console
$ git config wt.copymodified true
# or override for a single invocation
$ git wt --copymodified feature-branch
$ git wt --copymodified=false feature-branch  # explicitly disable
```

Default: `false`

#### `wt.nocopy` / `--nocopy`

Exclude files matching patterns from copying. Uses `.gitignore` syntax.

``` console
$ git config --add wt.nocopy "*.log"
$ git config --add wt.nocopy "vendor/"
# or override for a single invocation (multiple patterns supported)
$ git wt --copyignored --nocopy "*.log" --nocopy "vendor/" feature-branch
```

Supported patterns (same as `.gitignore`):
- `*.log`: wildcard matching
- `vendor/`: directory matching
- `**/temp`: match in any directory
- `/config.local`: relative to git root

#### `wt.copy` / `--copy`

Always copy files matching patterns, even if they are gitignored. Uses `.gitignore` syntax.

``` console
$ git config --add wt.copy "*.code-workspace"
$ git config --add wt.copy ".vscode/"
# or override for a single invocation (multiple patterns supported)
$ git wt --copy "*.code-workspace" --copy ".vscode/" feature-branch
```

This is useful when you want to copy specific IDE files (like VS Code workspace files) without enabling `wt.copyignored` for all gitignored files.

> [!NOTE]
> If the same file matches both `wt.copy` and `wt.nocopy`, `wt.nocopy` takes precedence.

> [!NOTE]
> The worktree base directory (`wt.basedir`) is always excluded from file copying, regardless of copy options. This prevents circular copying when basedir is inside the repository (e.g., `.worktrees/`).

#### `wt.hook` / `--hook`

Commands to run after creating a new worktree. Hooks run in the new worktree directory.

``` console
$ git config --add wt.hook "npm install"
$ git config --add wt.hook "go generate ./..."
# or override for a single invocation (multiple hooks supported)
$ git wt --hook "npm install" feature-branch
```

> [!NOTE]
> - Hooks only run when **creating** a new worktree, not when switching to an existing one.
> - If a hook fails, execution stops immediately and `git wt` exits with an error (shell integration will not `cd` to the worktree).

#### `wt.deletehook` / `--deletehook`

Commands to run before deleting a worktree. Hooks run in the worktree directory before it is removed, so you can perform cleanup (e.g., push branches).

``` console
$ git config --add wt.deletehook "git push origin --delete $(git branch --show-current)"
# or override for a single invocation (multiple hooks supported)
$ git wt -D --deletehook "npm run cleanup" feature-branch
```

> [!NOTE]
> - Hooks only run when deleting a **worktree**, not when deleting a branch without a worktree.
> - If a hook fails, execution stops immediately and the worktree is preserved.

#### `wt.nocd` / `--nocd`

Do not change directory to the worktree. Only print the worktree path.

Supported values for `wt.nocd` config:
- `true` or `all`: Never cd to worktree (both new and existing).
- `create`: Only prevent cd when creating new worktrees (allow cd to existing worktrees).
- `false` (default): Always cd to worktree.

``` console
# Prevent cd only for new worktrees (allow cd to existing)
$ git config wt.nocd create

# Never cd to any worktree
$ git config wt.nocd true

# Use --nocd flag for a single invocation (always prevents cd)
$ git wt --nocd feature-branch
```

> [!NOTE]
> - The `--nocd` flag always prevents cd regardless of config value.
> - Using `--nocd` with `--init` disables the `git()` wrapper entirely (only shell completion is output). The `wt.nocd` config does not affect `--init` output.

#### `wt.relative` / `--relative`

Append the current subdirectory path to the worktree output path. When running from a subdirectory, the output path will include the subdirectory relative to the repository root (like `git diff --relative`).

``` console
# Enable relative path resolution
$ git config wt.relative true

# Example: running from repo/some/path/
$ git wt feature-branch
/abs/.wt/feature-branch/some/path  # instead of /abs/.wt/feature-branch

# Use --relative flag for a single invocation
$ git wt --relative feature-branch
```

Default: `false`

> [!NOTE]
> If the subdirectory does not exist in the target worktree, the output falls back to the worktree root path.
