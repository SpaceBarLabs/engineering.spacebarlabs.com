---
layout: post
title:  "Setting up git crypt"
category: ""
date:   2024-09-20
---

We store some secrets in git repositories.  They are encrypted using [`git-crypt`](https://github.com/AGWA/git-crypt).  While setting it up, we had to refer to multiple tutorials and writeups, which are consolidated in this post.  We wrote this post for our own reference and hope it is helpful for the wider world as well.

## Installation

First, install `git-crypt` (on Ubuntu: `sudo apt install git-crypt`).  You can also refer to its [manpage](https://manpages.ubuntu.com/manpages/jammy/man1/git-crypt.1.html).

## Symmetric vs Asymmetric

There are two modes.  Symmetric and asymmetric.

[This article](https://dev.to/heroku/how-to-manage-your-secrets-with-git-crypt-56ih) sets up up a symmetric key, which would be a secret all users would need to share.

Instead, GPG can be used on a user-by-user basis.  It requires setting up a GPG key.  This is useful for GitHub and other tools as well.  Suggestion: use your work email when setting this up.

## How To

Make sure `gpg` is installed

See also: [How To Use GPG to Encrypt and Sign Messages by Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-use-gpg-to-encrypt-and-sign-messages)

### Generate a Key

Follow these steps ([original source](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)).


Accept defaults when prompted

```
gpg --full-generate-key
```

Then export the key:

```
gpg --list-secret-keys --keyid-format=long # Note the KEY_ID, e.g. ABCD1234EFGH5678
gpg --armor --export {KEY_ID}
```

### Upload the Key

Copy the whole block (including the "BEGIN" and the "END" lines) and add to your [GitHub GPG Keys](https://github.com/settings/keys) ([docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account)).

### Add the New User

Now, we're adding a new user.

A little-known fact about GitHub is that they make the public parts of SSH and GPG keys publicly available via their API.  Go ahead, try it with your username!

- SSH: [https://github.com/{username}.keys](https://github.com/{username}.keys)
- GPG: [https://github.com/{username}.gpg](https://github.com/{username}.gpg)

From a device that **already has the repository unlocked** (NOT the one you're adding):

```
curl https://github.com/{username}.gpg | gpg --import
```

Example output:

```
gpg: directory '/home/username/.gnupg' created
gpg: keybox '/home/username/.gnupg/pubring.kbx' created
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   651  100   651    0     0   6281      0 --:--:-- --:--:-- --:--:--  6259
gpg: /root/.gnupg/trustdb.gpg: trustdb created
gpg: key ABCD1234EFGH5678: public key "Other User <email@example.com>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

You may run into this error:

```
There is no assurance this key belongs to the named user
[stdin]: encryption failed: Unusable public key
```

[This SuperUser post](https://superuser.com/questions/1297502/gpg2-unusable-public-key-no-assurance-key-belongs-to-named-user) addresses that scenario and suggests:

    gpg --edit-key {KEY_ID}

Then at the sub-prompt:

    gpg> trust

If you're sure about the authenticity of the key, select trust level 5.

Add the key to `git-crypt`:

```
$ git-crypt add-gpg-user KEY_ID
```

(See the GitHub article or notes above for instructions on how to identify the `KEY_ID`, e.g `ABCD1234EFGH5678`.)

The `add-gpg-user` command creates a commit on the repository.  (If it didn't commit, something went wrong.)  **Remember to `git push` it.**

### Unlock the Repository on the New Device

Simply `git pull` and then `git-crypt unlock`
