# flox audit

`flox audit` traces a package in a Flox environment back to where it came from: the signed binary, the build recipe, the pinned nixpkgs commit, the source code, and what the project itself published. Each link is checked with a hash or a signature, and the output says in plain words what could and couldn't be checked.

```
binary on disk -> cache signature -> build recipe -> pinned nixpkgs commit -> source -> project's release signature
```

```sh
$ flox audit jq
...
Summary
   6 passed, 0 failed, 0 not checked, 0 can't check
```

## Requirements

- Flox with beta features turned on (see below). Tested with Flox 1.20.0.
- Nix 2.19 or newer on the host. `flox audit` uses the host's `nix` so it matches your Nix daemon. Tested on 2.31.6 (macOS and Linux) and 2.34.6 (Linux).

`jq`, `curl`, `git`, `gnupg`, `xz`, `coreutils` and `gnused` come with the package, so they don't need to be installed on the machine.

Tested on macOS (aarch64-darwin) and Ubuntu 24.04 (aarch64-linux).

## Install

Clone the repository and build it with Flox:

```sh
git clone https://github.com/jbayer/flox-audit
cd flox-audit
flox build
```

The build puts the command in `result-flox-audit/bin` inside your clone. Add that folder to your PATH:

```sh
export PATH="$PWD/result-flox-audit/bin:$PATH"
```

To have it in every new shell, add the same line to your shell profile (`~/.zshrc` or `~/.bashrc`) with the full path to your clone, for example:

```sh
export PATH="$HOME/src/flox-audit/result-flox-audit/bin:$PATH"
```

To update, pull and build again:

```sh
git pull
flox build
```

Changing into the clone may ask whether to auto-activate its Flox environment. You don't need to: the environment is only used to build `flox-audit`. Answer `n`, or run `flox activate deny` in the clone to stop being asked.

## Turn on beta features

Flox passes `flox audit` to the `flox-audit` command only when beta features are on. Without them you'll see:

```
✘ ERROR: no such command or positional: `audit`, did you mean `edit`?
```

Turn them on for your user:

```sh
flox config --set features.beta true
```

Or for a single command:

```sh
FLOX_FEATURES_BETA=true flox audit jq
```

You can also skip the subcommand and run `flox-audit jq` directly. That works with or without beta features.

To turn beta features off again, run `flox config --delete features`. Note that `flox config --delete features.beta` leaves `beta = false` in your config instead of removing it.

## Usage

```
flox audit [-d <dir> | -r <owner>/<name>] [--quick] [--no-key-fetch] <install-id>
```

| Option | What it does |
|---|---|
| `-d, --dir <dir>` | Environment directory. Defaults to the current directory. |
| `-r, --remote <owner>/<name>` | A FloxHub environment you have activated on this machine before. |
| `--quick` | Skip steps 5 to 7. These download nixpkgs and the package source, which takes longer. |
| `--no-key-fetch` | In step 7, don't look up signing keys online. Only keys in `FLOX_AUDIT_GPG_KEYS` are used. |
| `-h, --help` | Show help. |
| `-V, --version` | Show the version. |

`<install-id>` is the name of the package in your manifest's `[install]` section.

```sh
flox audit jq                              # environment in the current directory
flox audit -d ~/src/myapp python3          # another environment
flox audit -r jbayer/default flox-agent    # a FloxHub environment
flox audit --quick curl                    # signature and hash checks only
```

### Exit status

| Code | Meaning |
|---|---|
| 0 | Every check that ran passed |
| 1 | At least one check failed |
| 2 | Usage error, for example a package that isn't installed or a missing tool |

## What each step checks

### Packages from nixpkgs

1. **What did Flox record?** `.flox/env/manifest.lock` records the nixpkgs commit, the build recipe (`.drv`) and the output path for each system.
2. **Who built and signed it?** Asks cache.nixos.org and cache.flox.dev for their record of that output and shows who signed it.
3. **Are those signatures genuine?** `nix store verify` checks the cache's signature using only the public keys listed below. It also checks the copy on your machine.
4. **Has the copy on this machine been changed?** Nix re-hashes every file in the store path and compares it with the signed hash. A modified file fails here.
5. **Does the recorded recipe really produce this build?** Evaluates the package again from the pinned nixpkgs commit. The result has to produce the exact output path that was signed.
6. **Is the source code the same as when the recipe was written?** Downloads the source again and checks it against the hash in the recipe. Nix fails the download if the bytes don't match.
7. **Did the project publish this exact source code?** Looks next to the download for a signature or checksum the project published, and checks the source against it. See [Step 7](#step-7-the-projects-own-release) below.

### Packages published to a Flox catalog

Packages you or your team build and publish with Flox go through steps 1 to 4 the same way, using the local copy's signatures. Step 5 then shows the publisher's repository and commit, and checks that the commit can be fetched. Private repositories are checked if your saved git credentials can read them. `flox audit` never prompts for credentials.

A package published to a catalog was built on the publisher's machine or CI. To check it end to end, rebuild it from that commit and compare the output path.

## Step 7: the project's own release

Step 6 proves the source hasn't changed since the nixpkgs recipe was written. Step 7 asks a different question: is that source what the project itself released? It uses the strongest evidence it can find:

| Result | What it means |
|---|---|
| `PASS` with a key you chose | The project's signature is valid, using a key you supplied in `FLOX_AUDIT_GPG_KEYS`. The strongest check. |
| `PASS` with a key from the project's site | `flox audit` found the signing key on the project's own site (for example the GNU keyring), or the signer publishes the same key on their own domain through a Web Key Directory. Good evidence, but someone who controlled that site could replace both the download and the key. |
| `PASS (weak)` | The release is signed, but the key only came from a public key directory (keys.openpgp.org or keyserver.ubuntu.com), where anyone can upload a key under any name. It shows the file hasn't changed since it was signed, not who signed it. |
| `PASS`, "by a weaker check" | The source matches a checksum file the project posted on the same server. Catches mistakes, not a compromised server. |
| `FAIL` | The signature or checksum doesn't match, the project's signed checksum list doesn't include this file, or the signing key changed since an earlier audit. |
| `NOT CHECKED` | The release is signed but no key could be found, or the signature names only a short key ID, which isn't safe to look up. |
| `CAN'T CHECK` | The source is an archive GitHub generated (the project never published those exact bytes), or the project published no checksum or signature. |

Keys are only ever looked up by the full fingerprint in the signature, and a key is only used if its fingerprint matches exactly.

### Confirming keys yourself

To get the strongest result, download each project's key from its own website and pass it in:

```sh
curl -fsSLo ~/keys/curl.asc https://daniel.haxx.se/mykey.asc
FLOX_AUDIT_GPG_KEYS=~/keys/curl.asc flox audit curl
```

Separate several key files with `:`.

### Remembered keys

The first time `flox audit` sees a valid signature for a project, it saves the signing key's fingerprint in `~/.config/flox-audit/known-keys` (or `$FLOX_AUDIT_KNOWN_KEYS`). If a later release is signed with a different key, step 7 fails and shows both fingerprints. Projects do change keys, so check the project's announcements. If the change is expected, delete the old line from the file.

### How often each result appears

We sampled 105 common packages from nixos-25.11. About half (50) publish a signed release tarball. Without any keys supplied:

- 23 matched a key on the project's own site (all the GNU packages, through the GNU keyring)
- 1 matched through the signer's own domain (xz, via tukaani.org)
- 21 found the key only in a public directory (`PASS (weak)`), including curl, openssl, openssh, git, python3, zlib and nginx
- 5 can't be looked up safely because the signature names only a short key ID (cargo, rustc, file, flex, nmap)

Most of the rest are GitHub-generated archives, which step 7 can't check.

## Reading the output

**"cache first built it from … (lockfile recipe differs; same output, checked in step 5)"**

This is a note, not a failure. Two recipes can produce the same output when they differ only in how inputs are downloaded, such as the list of source URLs. The cache records whichever recipe it built first. Step 5 checks what matters: that your lockfile's recipe produces the exact output that was signed. You'll see this for `curl`, for example.

**"local copy has no signature from the keys above"**

Your Nix config may download from a mirror that signs with its own key. The store path and content hash are the same either way, and step 4 still re-hashes the contents.

## Trusted cache keys (steps 2 and 3)

`flox audit` checks binary cache signatures against these public keys:

```
cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY=
flox-cache-public-1:7F4OyH7ZCnFhcze3fJdfyXYLQw/aV7GEed86nQ7IsOs=
```

Check them against each publisher's documentation rather than trusting this file. To trust more keys, such as your own signing key, set `FLOX_AUDIT_EXTRA_KEYS`:

```sh
FLOX_AUDIT_EXTRA_KEYS="mycompany-1:..." flox audit my-tool
```

## Limits

- `-r` reads the lockfile Flox keeps for that environment under `~/.cache/flox/remote/`. Activate the environment once on the machine first.
- Steps 6 and 7 need the package to have a single `src`. Packages built from several sources show a note and skip the check.
- Step 7 can't detect a project whose own release was compromised by a maintainer. Signatures prove who released a file, not that the code is safe. The 2024 xz backdoor was in a properly signed release.
- `flox audit` checks one package at a time.
