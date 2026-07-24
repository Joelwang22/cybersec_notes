# Linux: Quick Command Guide

Commands are case-sensitive. Type them in a terminal and press Enter.

```bash
command [options] [arguments]
```

- `$` in examples means a normal-user prompt; do not type it.
- Press `Tab` to complete names and `Ctrl+C` to stop a running command.
- Use `man command` or `command --help` when you need details.

## Contents

- [Navigation](#navigation)
- [Files and directories](#files-and-directories)
- [Viewing and editing text](#viewing-and-editing-text)
- [Paths, wildcards, and quoting](#paths-wildcards-and-quoting)
- [Pipes and redirection](#pipes-and-redirection)
- [Finding files and text](#finding-files-and-text)
- [Permissions and ownership](#permissions-and-ownership)
- [Processes and jobs](#processes-and-jobs)
- [Installing software](#installing-software)
- [Archives and compression](#archives-and-compression)
- [Network commands](#network-commands)
- [System and storage](#system-and-storage)
- [Users and privileges](#users-and-privileges)
- [Shell shortcuts](#shell-shortcuts)
- [Common recipes](#common-recipes)
- [Safety](#safety)

<a id="navigation"></a>

## [Navigation](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show current directory | `pwd` |
| List files | `ls` |
| List details and hidden files | `ls -lah` |
| Enter a directory | `cd directory` |
| Go up one directory | `cd ..` |
| Go to your home directory | `cd` or `cd ~` |
| Go to the previous directory | `cd -` |
| Clear the terminal | `clear` or `Ctrl+L` |

<details>
<summary>Show example</summary>

```bash
pwd
ls -lah
cd Documents
cd ..
```

</details>

<a id="files-and-directories"></a>

## [Files and directories](#contents "Back to contents")

| Goal | Command |
|---|---|
| Create an empty file | `touch notes.txt` |
| Create a directory | `mkdir projects` |
| Create nested directories | `mkdir -p work/app/src` |
| Copy a file | `cp source.txt copy.txt` |
| Copy a directory | `cp -r source_dir destination_dir` |
| Move or rename | `mv old_name new_name` |
| Remove a file | `rm file.txt` |
| Remove an empty directory | `rmdir directory` |
| Remove a directory and contents | `rm -r directory` |
| Show file type | `file item` |
| Create a symbolic link | `ln -s target link_name` |

Use `-i` for confirmation before overwriting or deleting: `cp -i`, `mv -i`,
and `rm -i`.

<details>
<summary>Show example</summary>

```bash
mkdir -p project/docs
touch project/README.md
cp project/README.md project/docs/
mv project/docs/README.md project/docs/guide.md
```

</details>

<a id="viewing-and-editing-text"></a>

## [Viewing and editing text](#contents "Back to contents")

| Goal | Command |
|---|---|
| Print a short file | `cat file.txt` |
| Read a long file | `less file.txt` |
| Show first 10 lines | `head file.txt` |
| Show last 10 lines | `tail file.txt` |
| Follow a growing log | `tail -f app.log` |
| Count lines, words, and bytes | `wc file.txt` |
| Sort lines | `sort file.txt` |
| Remove adjacent duplicate lines | `uniq file.txt` |
| Edit with Nano | `nano file.txt` |
| Edit with Vim | `vim file.txt` |

In `less`, use arrow keys to move, `/text` to search, and `q` to quit. In
Nano, use `Ctrl+O` to save and `Ctrl+X` to exit.

<a id="paths-wildcards-and-quoting"></a>

## [Paths, wildcards, and quoting](#contents "Back to contents")

| Syntax | Meaning |
|---|---|
| `/` | Filesystem root |
| `~` | Your home directory |
| `.` | Current directory |
| `..` | Parent directory |
| `*` | Any number of characters |
| `?` | One character |
| `[abc]` | One listed character |

An absolute path starts at `/`; a relative path starts from the current
directory. Quote names containing spaces or special characters.

<details>
<summary>Show examples</summary>

```bash
cd /var/log
cd "My Documents"
ls *.txt
ls photo?.jpg
cp -- "-strange-name" backup
```

</details>

Use `--` before a filename beginning with `-` so it is not treated as an
option.

<a id="pipes-and-redirection"></a>

## [Pipes and redirection](#contents "Back to contents")

| Operator | Meaning |
|---|---|
| `command > file` | Write output to a file, replacing it |
| `command >> file` | Append output to a file |
| `command < file` | Read input from a file |
| `first \| second` | Send the first command's output to the second |
| `command 2> errors.log` | Write errors to a file |
| `command &> all.log` | Write output and errors to a file in Bash |
| `command \| tee file` | Display and save output |

<details>
<summary>Show examples</summary>

```bash
ls -lah > files.txt
printf '%s\n' "new line" >> notes.txt
ps aux | less
grep "ERROR" app.log | sort | uniq -c
```

</details>

<a id="finding-files-and-text"></a>

## [Finding files and text](#contents "Back to contents")

| Goal | Command |
|---|---|
| Find text in a file | `grep "text" file.txt` |
| Search directories recursively | `grep -R "text" directory` |
| Ignore case and show line numbers | `grep -in "text" file.txt` |
| Find by filename | `find directory -name "*.txt"` |
| Find files modified in the last day | `find directory -type f -mtime -1` |
| Locate a command | `command -v python` |
| Find files using an index | `locate filename` |

`rg` (ripgrep) is a fast alternative when installed: `rg "text" directory`.

<details>
<summary>Show examples</summary>

```bash
grep -Rin "TODO" src/
find . -type f -name "*.py"
find . -type f -size +100M
```

</details>

<a id="permissions-and-ownership"></a>

## [Permissions and ownership](#contents "Back to contents")

`ls -l` shows permissions such as `-rwxr-xr--`: read (`r`), write (`w`), and
execute (`x`) for the owner, group, and others.

| Goal | Command |
|---|---|
| Make a script executable | `chmod +x script.sh` |
| Set `rw-r--r--` | `chmod 644 file.txt` |
| Set `rwxr-xr-x` | `chmod 755 script.sh` |
| Change owner | `sudo chown user file` |
| Change owner and group recursively | `sudo chown -R user:group directory` |

Numeric values are `read = 4`, `write = 2`, and `execute = 1`. Add them for
each of owner, group, and others.

<a id="processes-and-jobs"></a>

## [Processes and jobs](#contents "Back to contents")

| Goal | Command |
|---|---|
| List processes | `ps aux` |
| Monitor processes | `top` |
| Find a process by name | `pgrep -af name` |
| Stop by process ID | `kill PID` |
| Force-stop as a last resort | `kill -9 PID` |
| Run in the background | `command &` |
| List shell jobs | `jobs` |
| Resume job in foreground | `fg %1` |
| Resume job in background | `bg %1` |

Press `Ctrl+Z` to pause the foreground process. Try normal `kill PID` before
`kill -9 PID`; normal termination allows cleanup.

For systems using systemd:

```bash
systemctl status service
sudo systemctl start service
sudo systemctl stop service
sudo systemctl restart service
journalctl -u service
```

<a id="installing-software"></a>

## [Installing software](#contents "Back to contents")

Use the commands for your distribution.

| Distribution | Refresh packages | Install | Remove |
|---|---|---|---|
| Ubuntu / Debian | `sudo apt update` | `sudo apt install package` | `sudo apt remove package` |
| Fedora | `sudo dnf check-update` | `sudo dnf install package` | `sudo dnf remove package` |
| Arch Linux | `sudo pacman -Syu` | `sudo pacman -S package` | `sudo pacman -R package` |

Upgrade Ubuntu or Debian packages with `sudo apt update && sudo apt upgrade`.
Review the proposed changes before confirming.

<a id="archives-and-compression"></a>

## [Archives and compression](#contents "Back to contents")

| Goal | Command |
|---|---|
| Create `.tar.gz` | `tar -czf archive.tar.gz directory` |
| Extract `.tar.gz` | `tar -xzf archive.tar.gz` |
| List archive contents | `tar -tf archive.tar.gz` |
| Create `.zip` | `zip -r archive.zip directory` |
| Extract `.zip` | `unzip archive.zip` |
| Compress one file | `gzip file` |
| Decompress one file | `gunzip file.gz` |

Extract an unfamiliar archive into a new empty directory first.

<a id="network-commands"></a>

## [Network commands](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show addresses and interfaces | `ip addr` |
| Test reachability | `ping example.com` |
| Fetch a URL | `curl -O URL` |
| Print a web response | `curl URL` |
| Download a file | `wget URL` |
| Show listening ports | `ss -tulpn` |
| Query DNS | `dig example.com` |
| Connect over SSH | `ssh user@host` |
| Copy to a remote host | `scp file user@host:/path/` |
| Synchronize directories | `rsync -av source/ destination/` |

`curl`, `wget`, `dig`, and `rsync` may need to be installed first.

<a id="system-and-storage"></a>

## [System and storage](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show kernel and system details | `uname -a` |
| Show distribution details | `cat /etc/os-release` |
| Show current user | `whoami` |
| Show date and time | `date` |
| Show memory usage | `free -h` |
| Show filesystem space | `df -h` |
| Show directory size | `du -sh directory` |
| Show largest items in a directory | `du -sh ./* \| sort -h` |
| List block devices | `lsblk` |
| Show command history | `history` |

<a id="users-and-privileges"></a>

## [Users and privileges](#contents "Back to contents")

| Goal | Command |
|---|---|
| Show identity and groups | `id` |
| Change your password | `passwd` |
| Run one command as administrator | `sudo command` |
| Switch user | `su - username` |
| Show logged-in users | `who` |

Use `sudo` only for commands that require it. Avoid running an entire terminal
as root.

<a id="shell-shortcuts"></a>

## [Shell shortcuts](#contents "Back to contents")

| Shortcut | Action |
|---|---|
| `Tab` | Complete a command or path |
| `Up` / `Down` | Browse command history |
| `Ctrl+C` | Stop the current command |
| `Ctrl+Z` | Pause the current command |
| `Ctrl+L` | Clear the screen |
| `Ctrl+A` / `Ctrl+E` | Move to start / end of line |
| `Ctrl+U` / `Ctrl+K` | Delete before / after cursor |
| `Ctrl+R` | Search command history |
| `!!` | Repeat the previous command |

Use `history` to see numbered commands and `!42` to rerun command 42. Check
the command carefully before pressing Enter.

<a id="common-recipes"></a>

## [Common recipes](#contents "Back to contents")

<details>
<summary>Show recipes</summary>

```bash
# Create a directory and enter it
mkdir project && cd project

# Run the next command only if the first succeeds
command1 && command2

# Run the next command only if the first fails
command1 || command2

# Count matching lines
grep -c "ERROR" app.log

# Show the ten largest items here
du -ah . | sort -h | tail -n 10

# Find recently changed files
find . -type f -mtime -1

# Copy a directory while preserving attributes
cp -a source destination

# Make and run a simple script
printf '%s\n' '#!/usr/bin/env bash' 'echo "Hello"' > hello.sh
chmod +x hello.sh
./hello.sh
```

</details>

<a id="safety"></a>

## [Safety](#contents "Back to contents")

- Read a command before running it, especially commands copied from the web.
- Check your location with `pwd` and targets with `ls` before `rm`, `mv`, or
  `chmod -R`.
- `rm` does not normally use a recycle bin. There is no general undo.
- Avoid `rm -rf` until you understand exactly what its path selects.
- `>` replaces a file; use `>>` to append.
- Quote variables in scripts: `"$filename"`.
- Keep backups of important data and test risky commands on disposable files.

<details>
<summary>Show help commands</summary>

```bash
man command
command --help
help shell_builtin
apropos "search words"
```

</details>
