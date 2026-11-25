# flox audit

`flox audit` traces a package in a Flox environment back to where it came from: the signed binary, the build recipe, the pinned nixpkgs commit, and the upstream source. Each link is checked with a hash or a signature. Nothing is taken on trust.

```
binary on disk -> cache signature -> build recipe -> pinned nixpkgs commit -> upstream source
```

```sh
$ flox audit jq
...
flox audit: all checks passed for jq
```

## Requirements

- Flox with beta features turned on (see below). Tested with Flox 1.20.0.
- Nix 2.19 or newer on the host. `flox audit` uses the host's `nix` so it matches your Nix daemon. Tested on 2.31.6 (macOS and Linux) and 2.34.6 (Linux).

`jq`, `curl`, `git`, `coreutils` and `gnused` come with the package, so they don't need to be installed on the machine.

Tested on macOS (aarch64-darwin) and Ubuntu 24.04 (aarch64-linux).

## Install

### Build from source

```sh
git clone https://github.com/jbayer/flox-audit
cd flox-audit
flox build
export PATH="$PWD/result-flox-audit/bin:$PATH"
```

### From a Flox catalog

Not published yet. Once it is:

```sh
flox install <owner>/flox-audit
```

Install it into your default environment to have `flox audit` available everywhere.

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
flox audit [-d <dir> | -r <owner>/<name>] [--quick] <install-id>
```

| Option | What it does |
|---|---|
| `-d, --dir <dir>` | Environment directory. Defaults to the current directory. |
| `-r, --remote <owner>/<name>` | A FloxHub environment you have activated on this machine before. |
| `--quick` | Skip steps 5 and 6. These download nixpkgs and the package source, which takes longer. |
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

1. **What the lockfile pins.** `.flox/env/manifest.lock` records the nixpkgs commit, the build recipe (`.drv`) and the output path for each system.
2. **Who signed it.** Asks cache.nixos.org and cache.flox.dev for their record of that output and shows who signed it.
3. **The signatures.** `nix store verify` checks the cache's signature using only the public keys listed below. It also checks the copy on your machine.
4. **The copy on your machine.** Nix re-hashes every file in the store path and compares it with the signed hash. A modified file fails here.
5. **The recipe.** Evaluates the package again from the pinned nixpkgs commit. The result has to match the lockfile's recipe and produce the exact output path that was signed.
6. **The upstream source.** Downloads the source again from upstream and checks it against the hash in the recipe. Nix fails the download if the bytes don't match. The sha256 is also printed in hex so you can compare it with the project's published checksum.

### Packages published to a Flox catalog

Packages you or your team build and publish with Flox go through steps 1 to 4 the same way, using the local copy's signatures. Step 5 then shows the publisher's repository and commit, and checks that the commit can be fetched. Private repositories are checked if your saved git credentials can read them. `flox audit` never prompts for credentials.

A package published to a catalog was built on the publisher's machine or CI. To check it end to end, rebuild it from that commit and compare the output path.

## Reading the output

**"cache first built it from … (lockfile recipe differs; same output, checked in step 5)"**

This is a note, not a failure. Two recipes can produce the same output when they differ only in how inputs are downloaded, such as the list of source URLs. The cache records whichever recipe it built first. Step 5 checks what matters: that your lockfile's recipe produces the exact output that was signed. You'll see this for `curl`, for example.

**"local copy has no signature from the keys above"**

Your Nix config may download from a mirror that signs with its own key. The store path and content hash are the same either way, and step 4 still re-hashes the contents.

## Trusted keys

`flox audit` checks signatures against these public keys:

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
- Step 6 needs the package to have a single `src`. Packages built from several sources show a note and skip the check.
- `flox audit` checks one package at a time.
