# mubit-cli

The command-line client for Mubit. Version 1.0.4.

Mubit learns from the model calls your agents already make. Send your OpenAI, Anthropic or Gemini calls through Mubit, and it:

- records each call and groups the calls into tasks;
- collects and rates training examples for each task;
- fine-tunes a smaller model for a task and tests it on examples it never trained on;
- serves a model that passes on a share of the task's traffic first, then on all of it, and rolls it back whenever you ask;
- forms lessons from the traffic.

You see and control all of this in the console at https://app.mubit.ai. `mubit-cli` lets you, your scripts and your CI do the same from a terminal.

## Install

### Homebrew

```bash
brew install MubitLabs/tap/mubit-cli
```

### Release binaries

Download the archive for your platform from the [latest release](https://github.com/MubitLabs/mubit-cli/releases/latest). A `.sha256` file sits beside each archive. Check the archive against it before you run the binary.

| Platform | Archive |
| --- | --- |
| macOS, Apple silicon | `mubit-cli-<version>-macos-arm64.tar.gz` |
| macOS, Intel | `mubit-cli-<version>-macos-x86_64.tar.gz` |
| Linux, arm64 | `mubit-cli-<version>-linux-arm64.tar.gz` |
| Linux, x86_64 | `mubit-cli-<version>-linux-x86_64.tar.gz` |
| Windows, x86_64 | `mubit-cli-<version>-windows-x86_64.zip` |

```bash
shasum -a 256 -c mubit-cli-<version>-<platform>.tar.gz.sha256
tar -xzf mubit-cli-<version>-<platform>.tar.gz
sudo mv mubit-cli-<version>-<platform>/mubit-cli /usr/local/bin/
```

On Windows, unzip the archive and put `mubit-cli.exe` in a folder on your `PATH`.

On Linux, the CLI keeps a stored key in the system secret service, so a downloaded build needs libdbus when it runs. Homebrew installs it for you.

```bash
sudo apt-get install -y libdbus-1-3    # Debian, Ubuntu
sudo dnf install -y dbus-libs          # Fedora, RHEL
```

Run `mubit-cli --version` to check the install, and `mubit-cli --help` for the list of commands.

## Sign in

`mubit-cli` signs in with a management key. It starts with `mwm_`. (Inference keys, which start with `mw_`, are for sending model calls through Mubit. The CLI doesn't use them.)

Only an admin of your organization can create a management key. If you aren't an admin, ask an admin of your organization to create one for you. The console shows a new key once, so copy it when it appears. If you lose a key, create a new one.

To create a management key:

1. Sign in at https://app.mubit.ai.
2. Open **All projects**. In the sidebar, under **Workspace**, select **Management keys**.
3. Select **Create management key**.
4. Enter a **Key name**, and choose a **Role**. The table below shows what each role can do.
5. Optional: choose a **Project scope** to limit the key to one project. The default is **All projects (organization-wide)**.
6. Select **Create management key** again. Select **Copy key**, save the key, and select **Done**.

| Role | What the key can do |
| --- | --- |
| `read` | Read everything except the list of management keys |
| `member` | Also rate calls, review training examples and import past calls |
| `admin` | Everything |

Then give the key to the CLI:

```bash
export MUBIT_MANAGEMENT_KEY=mwm_...     # read on every run; nothing is stored
mubit-cli whoami                        # your organization, and the key in use (masked)
```

To store the key instead, sign in once:

```bash
mubit-cli login                         # asks for the key without showing it, and stores it
mubit-cli whoami
```

In a script, pipe the key to `mubit-cli login` instead.

The key goes to your system keychain: macOS Keychain, Windows Credential Manager or the Linux secret service. On a machine without one, add `--storage file`. The key then goes to a file that only you can read. To make that the default, run `mubit-cli config set storage file`.

`mubit-cli logout` removes the stored key. The CLI looks for a key in this order: `--key`, then `MUBIT_MANAGEMENT_KEY`, then the stored key. The base URL is `https://api.mubit.ai`; change it with `--base-url` or `MUBIT_BASE_URL`.

## Manage Mubit from the terminal

With a management key, you can do everything you do in the console. The one exception is organization members, which you manage in the console.

```bash
mubit-cli tasks ls                                      # every task, with its stage
mubit-cli calls ls --project <project-id> --limit 20
mubit-cli calls feedback <call-id> --rating down --comment "Missed the sales tax line."

mubit-cli tasks train <task-id>                         # train a new version now

mubit-cli versions ls <task-id>
mubit-cli versions rollout <version-id> --canary 10     # serve it on 10% of the task's calls
mubit-cli versions promote <version-id>                 # then on all of them
mubit-cli versions rollback <version-id>                # or go back to the previous version
```

What you can do, by area:

### Organization and projects

| You can | Command |
| --- | --- |
| See your organization | `mubit-cli whoami` |
| List and open projects | `mubit-cli projects ls`, `mubit-cli projects get <id>` |
| Create, rename and delete a project | `mubit-cli projects create <name>`, `mubit-cli projects rename <id> <name>`, `mubit-cli projects delete <id>` |

### Keys

| You can | Command |
| --- | --- |
| List, create, rotate and revoke a project's inference keys | `mubit-cli keys ls --project <id>`, `mubit-cli keys create --project <id> --name <name>`, `mubit-cli keys rotate <id>`, `mubit-cli keys revoke <id>` |
| List, add, rotate and delete the provider keys Mubit calls providers with | `mubit-cli providers ls`, `mubit-cli providers add <provider>`, `mubit-cli providers rotate <id>`, `mubit-cli providers delete <id>` |
| List, create, rotate and revoke management keys | `mubit-cli management-keys ls`, `mubit-cli management-keys create --name <name> --role <role>`, `mubit-cli management-keys rotate <id>`, `mubit-cli management-keys revoke <id>` |

A provider key is the key Mubit calls a model provider with. `mubit-cli providers ls` lists them by provider id, such as `openai`, `anthropic`, `gemini` or `neon`. Only an admin of your organization can add, rotate or delete a provider key.

The Neon AI Gateway is the provider `neon`. A call on the Anthropic, OpenAI or Gemini route goes to the project's Neon key when the project has no key of its own for that provider.

To add a Neon key, sign in to the console at https://app.mubit.ai. Open **All projects**. In the sidebar, under **Workspace**, select **Provider keys**. Add a key, choose **Neon AI Gateway**, and enter two things:

- **Gateway URL**: the AI Gateway URL of your Neon branch. It is `https://` and the host, with no path, port or query. For example, `https://br-example-123456-api.ai.c-2.us-east-2.aws.neon.tech`.
- **Token**: a Neon token that starts with `nt_live_` and has the `ai_gateway:invoke` scope.

The list of providers shows **Neon AI Gateway** only where Neon is turned on for your organization. The region of a Neon key is always `global`. Mubit checks the token with Neon before it saves the key. To rotate a Neon key, enter a new token: the URL stays the key's own. To use another URL, delete the key and add it again.

### Calls

| You can | Command |
| --- | --- |
| List calls, filtered by project, task, model, status and time | `mubit-cli calls ls [--project <id>] [--task <id>] [--model <name>] [--status <status>] [--created-after <time>] [--created-before <time>]` |
| Open a call | `mubit-cli calls get <id>` |
| Rate a reply up or down, with a comment or a corrected answer | `mubit-cli calls feedback <id> --rating <up\|down> [--comment <text>] [--correction <text>]` |
| Delete everything stored for one session | `mubit-cli calls forget-session <session-id>` |

### Tasks

| You can | Command |
| --- | --- |
| List tasks, and open one: its stage, what it waits for and why | `mubit-cli tasks ls`, `mubit-cli tasks get <id>` |
| Read a task's timeline | `mubit-cli tasks timeline <id>` |
| Find out why a task isn't training or serving yet | `mubit-cli tasks diagnose <id>` |
| Review its training examples: keep or drop each one | `mubit-cli tasks examples <id>`, `mubit-cli tasks review <id> <example-id> --action <approve\|reject\|exclude>` |
| Train a new version now | `mubit-cli tasks train <id>` |
| Pause and resume a task's learning | `mubit-cli tasks pause <id>`, `mubit-cli tasks resume <id>` |
| Turn automatic routing to the task's trained models off or on | `mubit-cli tasks serving <id> <on\|off>` |
| Stop all of a task's trained models at once | `mubit-cli tasks stop <id> --reason <text>` |

### Trained models (versions)

| You can | Command |
| --- | --- |
| List a task's versions, and open one with its test results | `mubit-cli versions ls <task-id>`, `mubit-cli versions get <id>` |
| Run a version silently beside the current model, or serve it on a share of the calls | `mubit-cli versions rollout <id> [--shadow <percent>] [--canary <percent>]` |
| Promote, demote or roll back a version | `mubit-cli versions promote <id>`, `mubit-cli versions demote <id>`, `mubit-cli versions rollback <id>` |
| Read a task's promotion history | `mubit-cli tasks history <id>` |

### Settings

Your plan sets the outer limits. Within them, you can make a setting stricter or cheaper, but not looser.

| You can | Command |
| --- | --- |
| Read a project's settings, and who set each value | `mubit-cli settings get --project <id>` |
| Change the project settings you see in the console: approval mode, silent test, partial rollout, live checks, grading, when new versions train, the longest example, how long calls are kept and the pass bar | `mubit-cli settings set --project <id> <setting>=<value> ...` |
| Fine-tuning, per task: base model, retrain schedule, rows needed to train, the daily rating cap and the rating rubric | `mubit-cli tasks settings set <id> <setting>=<value> ...` |
| Routing, per task: when a trained model takes traffic, the silent-test and partial-rollout shares, the audit rate, and the quality bar it must hold | `mubit-cli tasks settings set <id> <setting>=<value> ...` |
| Lessons, per project: lesson injection on or off, its token budget, and the daily cap on reflections | `mubit-cli settings set --project <id> <setting>=<value> ...` |
| Spend: a daily cap on audit spend | `mubit-cli settings set --project <id> <setting>=<value> ...` |
| Read the audit of every settings change | `mubit-cli settings audit --project <id>` |
| Read a task's settings | `mubit-cli tasks settings get <id>` |

### Lessons, imports, usage and activity

| You can | Command |
| --- | --- |
| List and open what Mubit learned | `mubit-cli lessons ls`, `mubit-cli lessons get <id>` |
| Import past calls from Langfuse, Braintrust or Logfire, follow the import, or cancel it | `mubit-cli imports create --source <source>`, `mubit-cli imports ls`, `mubit-cli imports cancel <id>` |
| See usage: calls and tokens | `mubit-cli usage` |
| See your balance and credit | `mubit-cli billing` |
| Read a project's activity | `mubit-cli activity` |
| Follow a long-running job, such as a training run or an import | `mubit-cli jobs get <id>`, `mubit-cli jobs wait <id> [--timeout <seconds>]` |

`mubit-cli jobs wait` prints the job's last state when it stops. It exits 0 when the job succeeds, and 1 when the job fails, is canceled, or the timeout (60 seconds unless you set another) passes first.

## Output and scripting

- **Tables by default.** `--json` prints the API's response as JSON, and `--jsonl` prints one JSON object per line. Progress, warnings and errors go to stderr, so stdout stays pipeable.
- **Lists come in pages.** A list command prints its first page. When there are more, a table ends with `Next cursor: <value>`, and `--json` shows it as `next_cursor`. Pass it back with `--cursor <value>`. `--limit <n>` sets the page size.
- **Destructive commands ask first**, for example `versions rollback` and `management-keys revoke`. In a script, pass `--yes`. Without a terminal and without `--yes`, such a command makes no request and exits with code 2.
- **A new key is shown once**, when you create or rotate it. Save it then.

| Flag | Environment variable | What it does |
| --- | --- | --- |
| `--key` | `MUBIT_MANAGEMENT_KEY` | The management key to use. Prefer the variable: a key on the command line shows up in shell history and process lists. |
| `--base-url` | `MUBIT_BASE_URL` | The API's address. The default is `https://api.mubit.ai`. |
| `--project` | `MUBIT_PROJECT` | The project for commands that filter by project. |
| `--storage` | `MUBIT_CLI_STORAGE` | Where `login` keeps the key: `keyring` (the default) or `file`. |
| `--json`, `--jsonl` | `MUBIT_CLI_OUTPUT` (`json` or `jsonl`) | The output format. `mubit-cli config set output json` makes JSON the default. |
| `--yes` | | Skip the confirmation question. |

## Exit codes

| Code | Meaning |
| --- | --- |
| 0 | success |
| 1 | unexpected error, or a job that failed, was canceled or timed out under `jobs wait` |
| 2 | usage error: a bad flag or value, an HTTP 400, or confirmation required (re-run with `--yes`) |
| 3 | no key, or the key was refused (HTTP 401 or 403) |
| 4 | not found (HTTP 404) |
| 5 | conflict (HTTP 409 or 412) |
| 6 | rate limited (HTTP 429) |
| 7 | server error (HTTP 5xx) |

With `--json`, an error also prints a JSON object with `error` and `exit_code` on stderr.

## Related

- **Console:** https://app.mubit.ai
- **Python client:** `pip install mubit-sdk`
- **JavaScript client:** `npm install @mubit-ai/sdk`
