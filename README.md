# CredsHunter
<div align="center">

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/19a115de-0fc1-4745-ba61-b3a69994d07a" />

<br>

![read-only](https://img.shields.io/badge/read--only-yes-3fb950?style=flat-square)
![no network](https://img.shields.io/badge/network-none-3fb950?style=flat-square)
![bash](https://img.shields.io/badge/bash-4%2B-2b3137?style=flat-square&logo=gnubash&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-2b3137?style=flat-square&logo=powershell&logoColor=white)
![authorized use only](https://img.shields.io/badge/use-authorized%20only-f85149?style=flat-square)

</div>

`credshunter` is a read-only credential finder for authorized post-exploitation, CTFs, labs, and internal assessments. It walks one or more paths and surfaces reusable secrets, encrypted credential leads, private keys, credential containers, and files that point to where credentials may be hiding.

Two siblings, one behavior: `credshunter.sh` for Linux and `credshunter.ps1` for Windows.

## How It Works

A staged funnel narrows from known credential locations to suspicious files and content-level matches. The tool does not authenticate, spray, dump processes, exploit services, change files, or touch the network.

```text
 Stage 1   OS and user artifacts      registry, services, histories, vaults, sessions, shortcuts, workspaces
 Stage 2   Credential containers      kdbx, ppk, pfx, keytab, axx, enc, gpg, renamed archives
 Stage 3   High-value file types      keys, env files, backups, DBs, captures, configs
 Stage 4   Suspicious filenames       password, secret, credential, backup, vault
 Stage 5   Content scan               tuned regexes for reusable credentials and encrypted leads
```

The newer reference-led checks also inspect files that commonly reveal where credentials live, such as shell histories, PowerShell history, VS Code workspaces, Sublime sessions, recent shortcuts, and similar user artifacts.

## What It Finds

CredsHunter focuses on local, reusable, or investigation-worthy credential material:

| Category | Examples |
|---|---|
| Direct credentials | `password=...`, connection strings, basic auth URLs, Windows service command lines, WinRM/Impacket commands, PHP arrays, PHP `define()`, DB connect calls |
| Private keys and auth material | SSH keys, PuTTY keys, PFX/P12, keytabs, SAM/SYSTEM hives, GPP `cpassword` |
| Credential containers | KeePass `.kdbx`, encrypted archives, `.axx`, `.enc`, `.gpg`, `.pgp`, renamed ZIP/7z/RAR/GZip/TAR files |
| Encrypted credential leads | Ansible Vault, SOPS encrypted values, Kubernetes SealedSecret, Kubernetes `encryptedData` |
| Reference leads | Paths found in histories, recent files, shortcuts, editor sessions, and workspace files |
| Interesting files | `.env`, backups, configs, database dumps, captures, likely credential filenames |

Cloud and SaaS API token hunting is intentionally limited to reduce noise. Local cloud CLI credential files may still be listed as interesting artifacts.

On Windows, Stage 1 enumerates service `ImagePath`, `ObjectName`, and string values in each service's `Parameters` subkey. Literal passwords in command-line arguments, URLs, or password-named registry values are reported as `[HIGH]`; services running under non-built-in accounts are listed for review. Passwords managed normally by the Service Control Manager are stored as protected LSA secrets, so the tool does not attempt to extract them.

## Usage

Linux:

```bash
# Full filesystem sweep, verbose console, save everything for later
./credshunter.sh -p / -o loot.txt

# Cleaner console summary while still exporting to a file
./credshunter.sh -p / --clean -o loot.txt

# Target common CTF/user locations
./credshunter.sh -p /home -p /var/www -p /opt --clean -o loot.txt

# Targeted scan, skip content scanning
./credshunter.sh -p /var/www -p /home --no-stage5

# Pipe-friendly output
./credshunter.sh -p /home --clean --no-color -o loot.txt
```

Windows:

```powershell
# Full C: sweep, verbose console, save everything for later
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 -Path C:\ -OutputFile .\loot.txt

# Cleaner console summary while still exporting to a file
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 -Path C:\ -Clean -OutputFile .\loot.txt

# Target common user/app locations
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 -Path C:\Users -Path C:\inetpub -Clean -NoColor -OutputFile .\loot.txt

# Web / DB box: also scan SQL and CSV dumps
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 -Path D:\ -IncludeData
```

You do not need to be privileged to run it. Running as Administrator or root only increases what the OS allows the script to read.

## Output Modes

Default mode prints findings as each stage finishes. This is useful when you want full visibility during a scan, but it can be noisy on real systems.

Clean mode keeps the console focused:

```bash
./credshunter.sh -p /home --clean -o loot.txt
```

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 -Path C:\Users -Clean -OutputFile .\loot.txt
```

In clean mode, the final console output is grouped into:

```text
Directly usable credentials
Credential containers
Private keys and auth material
Encrypted credential leads
References and user-artifact leads
Other interesting files
Counts
```

`-o` / `-OutputFile` still works with clean mode, so you can keep a file for later review while keeping the terminal readable.

Use `--no-color` / `-NoColor` when redirecting output, pasting results into reports, running in limited terminals, or processing with tools like `grep`, `findstr`, or `Select-String`.

## Output Tags

Findings are grouped by severity and type:

| Tag | Meaning |
|---|---|
| `[CRITICAL]` | Confirmed credential container or highly sensitive credential artifact |
| `[HIGH]` | Reusable password, hash, connection string, GPP cpassword, command-line credential |
| `[KEY]` | Private key or readable auth material such as SAM/SYSTEM hives |
| `[ENCRYPTED_CREDENTIAL_LEAD]` | Encrypted secret block that needs a password/key to decrypt |
| `[CREDENTIAL_LEAD]` | Artifact likely to contain or point to credential material |
| `[REFERENCE]` | Path reference found in user artifacts, histories, editor sessions, or shortcuts |
| `[INTEREST]` | High-value file worth reviewing |
| `[NAME]` | Suspicious filename review hint |

The exit code is `1` whenever anything lands in `CRITICAL`, `HIGH`, or `KEY`; otherwise it exits `0`.

## Tuning

| Want to... | Linux | Windows |
|---|---|---|
| Limit scope | `-p /path` | `-Path C:\Path` |
| Exclude a path | `-x /path` | `-ExcludePath C:\Path` |
| Clean summary | `--clean` | `-Clean` |
| Disable colors | `--no-color` | `-NoColor` |
| Save output | `-o loot.txt` | `-OutputFile .\loot.txt` |
| Scan every file | `-a` / `--all` | `-All` |
| Change size cap | `-m N` / `--max-file-size-mb N` | `-MaxFileSizeMB N` |
| Disable size cap | `--no-size-limit` | `-NoSizeLimit` |
| Skip a stage | `--no-stageN` | `-NoStageN` |
| Include SQL/CSV dumps | default set plus options | `-IncludeData` |

Patterns and file-type lists live in clearly labeled arrays near the top of each script.

## FAQ

**Does it change anything on the host?**  
No. It is read-only, writes only to the output file you choose, never touches the network, and exits cleanly on Ctrl-C.

**Why can a full disk scan take a long time?**  
Full disk scans walk a lot of files and Stage 5 reads file content. In CTFs and labs, scans are usually faster because the target paths are smaller. For speed, scan likely locations first: `/home`, `/var/www`, `/opt`, `C:\Users`, `C:\inetpub`, app folders, backup folders, and web roots.

**What does an encrypted credential lead mean?**  
It means CredsHunter found something that probably contains a secret, but the value is encrypted. Examples include Ansible Vault, SOPS, and SealedSecret data. The report shows the file and marker so you know where to focus next.

**Why use clean mode?**  
Clean mode is better for quick triage and reports. Default mode is better when you want to watch each stage and see every raw finding as it appears.

**Why use no-color?**  
Colors are helpful in an interactive terminal, but ANSI color codes can be annoying in redirected output, report snippets, or text processing. `--no-color` / `-NoColor` keeps the output plain.

**A password was missed. What should I check?**  
Confirm the target path is included, the file is under the size cap, the path is not excluded, and the extension is scanned by Stage 5. Use `-All` / `--all` for broader content scanning.

## Wiki

Check out the [Wiki Document](https://github.com/NeCr00/Credential-Hunting/wiki) for more information about the project.

## Contribute

Feel free to contribute to the project.
