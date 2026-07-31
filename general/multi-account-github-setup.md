# Managing multiple GitHub accounts on one machine

Two independent pieces make this work together: **SSH keys** (which account you authenticate
as when pushing/pulling) and **git identity** (whose name/email goes on the commits you make).
They're separate systems — get both right, in either order.

---

## 1. SSH keys — one per GitHub account

Each GitHub account needs its own SSH key pair, added to that account's own
**Settings → SSH and GPG keys**. Never reuse one key across two different GitHub accounts.

### Current setup on this machine

```
~/.ssh/id_ed25519           # default/work key — used by the "github-work" alias
~/.ssh/id_ed25519_personal  # sparjapati key   — used by the "github-personal" alias
```

### `~/.ssh/config`

This is what actually routes each alias to its own key:

```
Host github-work
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
  IdentityFile ~/.ssh/id_ed25519

Host github-personal
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentitiesOnly yes
  IdentityFile ~/.ssh/id_ed25519_personal
```

`IdentitiesOnly yes` is the important bit — without it, SSH will try every key loaded in your
agent in turn, and might authenticate as the wrong account before ever trying the right key.

### Using an alias instead of `github.com`

Clone and set remotes using the **alias**, not `github.com`, directly:

```bash
git clone git@github-personal:sparjapati/some-repo.git
# or, for an existing repo:
git remote set-url origin git@github-personal:sparjapati/some-repo.git
```

The alias name (`github-personal`) is arbitrary — SSH just needs it to match a `Host` block; it
has nothing to do with the actual GitHub username or org.

### Adding a new account later

```bash
ssh-keygen -t ed25519 -C "<email for the new account>" -f ~/.ssh/id_ed25519_<name>
```

Then add a new `Host github-<name>` block above pointing `IdentityFile` at it, add the matching
`.pub` file's contents to that GitHub account's SSH keys settings, and clone/set-url using
`git@github-<name>:...`.

### Verifying which account a key authenticates as

```bash
ssh -T git@github-personal
# -> "Hi sparjapati! You've successfully authenticated..."
ssh -T git@github-work
# -> "Hi <work-username>! You've successfully authenticated..."
```

---

## 2. Git identity (name/email) — scoped by directory

SSH decides *which account you can push as*; it does **not** set the name/email that goes on
your commits. That's a separate git config concern, and it's easy to end up with commits
authored under the wrong email even when pushing through the right SSH key. Fix this with a
conditional include in `~/.gitconfig`, keyed on the repo's directory:

### `~/.gitconfig`

```ini
[user]
	name = Sanjay Verma
	email = sanjay.verma@oxyzo.in

[includeIf "gitdir:~/Documents/coding/"]
	path = ~/.gitconfig-coding
```

The top-level `[user]` block is the **default** — anything not matched by an `includeIf` below
it (e.g. `~/Desktop/oxyzofinance/*` projects) uses this.

### `~/.gitconfig-coding`

```ini
[user]
	name = sparjapati
	email = parjapatsanjay1999@gmail.com
```

Any repo whose `.git` directory lives under `~/Documents/coding/` picks up this identity instead
— no per-repo `git config user.email` needed, and no risk of forgetting to set it in a new repo
you create there.

### Adding another directory-scoped identity later

Add another `includeIf` block to `~/.gitconfig`, pointing at its own separate file:

```ini
[includeIf "gitdir:~/Desktop/some-other-tree/"]
	path = ~/.gitconfig-some-other-tree
```

`includeIf "gitdir:..."` matching is prefix-based on the actual repository path (with `~`
expanded to `$HOME`) — it doesn't care about the remote URL or which SSH key you're using, only
where the repo sits on disk.

### Verifying which identity is active

```bash
cd /path/to/some/repo
git config user.name
git config user.email
```

---

## Putting both together

For a repo under `~/Documents/coding/`:
- **Commits** are authored as `sparjapati <parjapatsanjay1999@gmail.com>` (via the `includeIf`).
- **Push/pull** authenticates as whichever GitHub account owns the key behind the alias in the
  remote URL (`git@github-personal:...` → the `sparjapati` account, assuming that's the key
  registered there).

These two don't have to match the same account by construction — git will happily let you push
commits authored as one identity using SSH credentials for a completely different account. It's
on you to keep the directory-scoped git identity and the SSH alias/key pointed at the same
logical account when that's the intent (as it is here: both point at `sparjapati`).
