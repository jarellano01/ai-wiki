# SSH Setup for GitHub (macOS)

SSH lets you push and pull from GitHub without entering your username and password every time. Instead, your computer uses a cryptographic key pair — a private key (stays on your machine, never share it) and a public key (you give to GitHub).

## 1. Check for existing SSH keys

```bash
ls -al ~/.ssh
# ↑ Lists files in your .ssh directory. If you see id_ed25519 and id_ed25519.pub
#   (or id_rsa and id_rsa.pub), you already have a key pair and can skip to step 3.
#   If the folder doesn't exist or is empty, continue to step 2.
```

## 2. Generate a new SSH key

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
# ↑ Creates a new SSH key pair using the ed25519 algorithm (modern, fast, secure).
#   The -C flag adds your email as a label so you can identify the key later.
#   Use the same email you use for your GitHub account.
```

You'll see a series of prompts:

```
Enter file in which to save the key (/Users/you/.ssh/id_ed25519):
```
**Press Enter** to accept the default location.

```
Enter passphrase (empty for no passphrase):
```
**Type a passphrase** (recommended) or press Enter to skip. A passphrase adds an extra layer of security — even if someone gets your key file, they can't use it without the passphrase. You won't have to type it every time (the next step handles that).

## 3. Add the key to your Mac's keychain

```bash
eval "$(ssh-agent -s)"
# ↑ Starts the SSH agent — a background process that holds your keys in memory
#   so you don't have to type your passphrase repeatedly. You'll see output like
#   "Agent pid 12345" which means it's running.
```

Create or edit the SSH config file to automatically load your key and store the passphrase in macOS Keychain:

```bash
cat >> ~/.ssh/config << 'EOF'
Host github.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
EOF
# ↑ This tells SSH: "When connecting to github.com, use this key file and
#   save the passphrase in macOS Keychain so I never have to type it again."
```

> If `~/.ssh/config` already exists and has a `Host github.com` block, edit it instead of appending.

Now add the key to the agent:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
# ↑ Loads your private key into the SSH agent and stores the passphrase in
#   macOS Keychain. After this, SSH connections to GitHub "just work" —
#   even after restarting your Mac.
```

## 4. Add the public key to GitHub

Copy the public key to your clipboard:

```bash
pbcopy < ~/.ssh/id_ed25519.pub
# ↑ Copies the contents of your PUBLIC key to the clipboard.
#   This is the key you share — it's safe to give to GitHub.
#   Never share id_ed25519 (without .pub) — that's your private key.
```

Then add it to GitHub:

1. Go to [github.com/settings/keys](https://github.com/settings/keys)
2. Click **New SSH key**
3. **Title**: something descriptive like "MacBook Pro" or "Work Laptop"
4. **Key type**: Authentication Key
5. **Key**: paste what you copied (Cmd+V)
6. Click **Add SSH key**

## 5. Test the connection

```bash
ssh -T git@github.com
# ↑ Attempts an SSH connection to GitHub. You're not actually logging in —
#   GitHub's SSH server just verifies your key and responds.
```

The first time you connect, you'll see:

```
The authenticity of host 'github.com (...)' can't be established.
ED25519 key fingerprint is SHA256:+DiY...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes`. This adds GitHub to your known hosts file so you won't see this again.

If everything worked, you'll see:

```
Hi YOUR_USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

That means SSH is set up and ready to go.

## Using SSH URLs with git

Once SSH is configured, use SSH URLs instead of HTTPS when working with repos:

```bash
# HTTPS (requires a token or password for private repos)
git clone https://github.com/owner/repo.git

# SSH (uses your key — works automatically)
git clone git@github.com:owner/repo.git
```

When creating repos or adding remotes, use the SSH URL:

```bash
git remote add origin git@github.com:YOUR_USERNAME/my-project.git
```

> **How to tell which URL your repo uses:** Run `git remote -v`. If it shows `https://`, you can switch to SSH with:
> ```bash
> git remote set-url origin git@github.com:YOUR_USERNAME/my-project.git
> ```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `Permission denied (publickey)` | Your key isn't loaded. Run `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` and verify the public key is added on [github.com/settings/keys](https://github.com/settings/keys). |
| `Could not open a connection to your authentication agent` | The SSH agent isn't running. Run `eval "$(ssh-agent -s)"` first. |
| Key works but you still get password prompts | Your remote is using HTTPS, not SSH. Run `git remote -v` to check, and switch with `git remote set-url origin git@github.com:owner/repo.git`. |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Key file permissions are too open. Fix with `chmod 600 ~/.ssh/id_ed25519`. |
| Passphrase prompt every time | The keychain entry may be missing. Run `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` again. |
