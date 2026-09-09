# mubit-cli

Prebuilt binaries for `mubit-cli`, the command-line client for the Mubit console. Use it to manage organizations, projects, instances, runtime users, keys, and usage from a terminal.

This repository carries release artifacts only. It holds no source code.

## Install with Homebrew

```bash
brew install mubit-ai/tap/mubit-cli
```

## Install manually

Download the archive for your platform from the [latest release](https://github.com/mubit-ai/mubit-cli/releases/latest), then verify it against the `.sha256` file published beside it:

```bash
shasum -a 256 -c mubit-cli-<version>-<platform>.tar.gz.sha256
tar -xzf mubit-cli-<version>-<platform>.tar.gz
sudo mv mubit-cli-<version>-<platform>/mubit-cli /usr/local/bin/
```

Verify the checksum before you run the binary.

| Platform | Archive |
|---|---|
| macOS, Apple silicon | `macos-arm64.tar.gz` |
| macOS, Intel | `macos-x86_64.tar.gz` |
| Linux, arm64 | `linux-arm64.tar.gz` |
| Linux, x86_64 | `linux-x86_64.tar.gz` |
| Windows, x86_64 | `windows-x86_64.zip` |

On Linux the binary reads and writes credentials through the D-Bus secret service, so it needs `libdbus`. Homebrew installs this for you. For a manual install:

```bash
sudo apt-get install -y libdbus-1-3    # Debian/Ubuntu
sudo dnf install -y dbus-libs          # Fedora/RHEL
```

Use `--storage file` to keep credentials in a `0600` JSON file instead of the OS keychain. This suits headless machines.

## Getting started

```bash
mubit-cli login       # opens a browser; approve, then return to the terminal
mubit-cli whoami      # identity, organization, session expiry
mubit-cli projects list
```

On a remote machine without a browser:

```bash
mubit-cli login --device
```

Login issues an organization-bound management session. It provisions nothing and it mints no runtime key. Each release archive contains a `README.md` with the full command reference, the exit codes, and the configuration keys.

## Reporting a problem

Open an issue on this repository. Include the command you ran, the exit code, and the version from `mubit-cli --version`. Never include an API key or a token.
