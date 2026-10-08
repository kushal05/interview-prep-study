# Bash & Shell Scripting

> **TL;DR:** Every production shell script should start with `set -euo pipefail`, quote its variables, and trap its exits. Most outages from scripts come from the missing `"`.

## The Mandatory Header

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'                # safer field splitting (no spaces)

# -e : exit on first error
# -u : error on unset variable
# -o pipefail : pipeline fails if any stage fails (not just the last)
```

Why? Without `-e`, a failing `cp` continues to `rm`. Without `pipefail`, `curl ... | tee log` reports success even if `curl` returned 500.

## Variables & Quoting

```bash
name="kushal"
greeting="Hello, $name"            # double quotes interpolate
literal='Hello, $name'             # single quotes are literal

# ALWAYS quote variables to handle spaces / globs
file="my file.txt"
rm "$file"                         # good
rm $file                           # bad — tries to delete "my" and "file.txt"

# Default values
: "${PORT:=8080}"                  # set PORT to 8080 if unset/empty
: "${REGION:?must be set}"         # exit with error if unset
: "${LOG_DIR:-/var/log}"           # use /var/log if unset, don't assign
```

### String operations (no `sed` needed)

```bash
path="/var/log/nginx/access.log"
echo "${path##*/}"                 # access.log    (basename)
echo "${path%/*}"                  # /var/log/nginx (dirname)
echo "${path%.log}.gz"             # /var/log/nginx/access.gz (replace ext)
echo "${path/log/LOG}"             # first replace
echo "${path//log/LOG}"            # global replace
echo "${#path}"                    # length
echo "${path:5:8}"                 # substring start at 5, len 8
```

## Conditionals

```bash
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "config present"
elif [[ -d /etc/nginx ]]; then
    echo "dir exists, no config"
else
    echo "nginx not installed"
fi

# Numeric vs string comparison
[[ "$count" -gt 10 ]]              # numeric
[[ "$name" == "prod" ]]            # string
[[ "$name" == prod* ]]             # glob
[[ "$name" =~ ^prod-[0-9]+$ ]]     # regex

# Combinators
[[ -f "$f" && -r "$f" ]]
[[ -z "$var" ]]                    # empty
[[ -n "$var" ]]                    # non-empty

# Short-circuit (one-liners)
command_a && command_b             # b runs only if a succeeded
command_a || command_b             # b runs only if a failed
command_a || { echo "fail"; exit 1; }
```

### File tests

| Test | Meaning |
|------|---------|
| `-e FILE` | exists |
| `-f FILE` | regular file |
| `-d DIR` | directory |
| `-L FILE` | symlink |
| `-r/-w/-x` | readable/writable/executable |
| `-s FILE` | non-empty |
| `FILE1 -nt FILE2` | newer than |

## Loops

```bash
# Over args
for arg in "$@"; do echo "arg: $arg"; done

# Over file lines (correct way)
while IFS= read -r line; do
    echo "line: $line"
done < input.txt

# Over command output — beware!
# Wrong (breaks on spaces): for f in $(ls); do ...
# Right:
find . -name "*.log" -print0 | while IFS= read -r -d '' f; do
    gzip "$f"
done

# C-style
for ((i=0; i<10; i++)); do echo "$i"; done

# Range
for i in {1..5}; do echo "$i"; done
for i in {0..100..10}; do echo "$i"; done   # step

# Until
until curl -fs http://localhost:8080/health > /dev/null; do
    sleep 2
done
```

## Functions

```bash
log() {
    local level="$1"; shift        # shift removes $1
    local msg="$*"
    printf '[%s] %s %s\n' "$(date -Iseconds)" "$level" "$msg" >&2
}

retry() {
    local max=$1; shift
    local i=0
    until "$@"; do
        i=$((i+1))
        [[ $i -ge $max ]] && return 1
        sleep $((2**i))            # exponential backoff
    done
}

# usage
retry 5 curl -fsS https://api.example.com/health
log INFO "deployment complete"
```

Always declare locals with `local` — otherwise they leak to the global scope.

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Generic error |
| 2 | Misuse of shell builtin |
| 126 | Command found but not executable |
| 127 | Command not found |
| 128 + N | Killed by signal N (130 = Ctrl+C / SIGINT) |
| 255 | Out of range |

```bash
do_thing
rc=$?                              # capture before any other command
if [[ $rc -ne 0 ]]; then
    log ERROR "do_thing failed with $rc"
    exit "$rc"
fi
```

## Traps — Cleanup That Always Runs

```bash
tmpdir=$(mktemp -d)
cleanup() {
    local rc=$?
    rm -rf "$tmpdir"
    [[ $rc -ne 0 ]] && log ERROR "script failed with rc=$rc"
    exit "$rc"
}
trap cleanup EXIT INT TERM

# Stack traps for finer control
trap 'echo "interrupted by user"; exit 130' INT
```

## Useful One-Liners

```bash
# Top 10 IPs in access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Top 10 slowest URLs (assuming time is field $7)
awk '{print $7, $11}' access.log | sort -rn | head

# Disk hogs
du -ah /var | sort -rh | head -20

# Find files modified in last 1 day
find /var/log -type f -mtime -1

# Delete files older than 30 days
find /backups -type f -mtime +30 -delete

# Process JSON without jq panic
curl -s api/users | jq -r '.[] | "\(.id)\t\(.email)"'

# Bulk rename
for f in *.JPG; do mv "$f" "${f%.JPG}.jpg"; done

# Get external IP
curl -s ifconfig.me

# Tail multiple logs together
tail -f /var/log/app/*.log

# Wait for a port to open
until nc -z localhost 5432; do sleep 1; done
```

## Argument Parsing — getopts

```bash
usage() { echo "usage: $0 -e ENV [-r REGION] [-v]"; exit 1; }

env=""
region="us-east-1"
verbose=0

while getopts "e:r:vh" opt; do
    case "$opt" in
        e) env="$OPTARG" ;;
        r) region="$OPTARG" ;;
        v) verbose=1 ;;
        h|*) usage ;;
    esac
done
shift $((OPTIND-1))

[[ -z "$env" ]] && usage
```

## Heredocs & Process Substitution

```bash
# Heredoc — useful for SQL, config templates
psql <<-SQL
    CREATE TABLE IF NOT EXISTS t (id int);
    INSERT INTO t VALUES (1);
SQL

# Quoted heredoc — no $variable expansion
cat > /etc/nginx/conf.d/app.conf <<'EOF'
server { listen 80; server_name ${HOST}; }   # literal
EOF

# Process substitution
diff <(sort file1) <(sort file2)
```

## Interview Questions

**Q: Why `set -euo pipefail`?**
A: `-e` aborts on error so failures don't cascade. `-u` catches typos in variable names. `pipefail` makes `cmd1 | cmd2` fail if `cmd1` fails (otherwise only `cmd2`'s exit code matters).

**Q: What does `$@` vs `$*` do?**
A: Both expand to all positional args. `"$@"` expands each arg as a separate quoted word (correct in 99% of cases). `"$*"` joins them with the first char of `IFS`.

**Q: How do you debug a bash script?**
A: `bash -x script.sh` prints each command before execution. Or `set -x` / `set +x` to toggle within the script. `PS4='+ ${BASH_SOURCE}:${LINENO}: '` adds file:line prefix.

**Q: Difference between `>` and `>>` and `2>&1`?**
A: `>` overwrites stdout, `>>` appends. `2>&1` redirects stderr to wherever stdout is going. `&> file` does both (bash-only).

**Q: What does `trap '...' EXIT` give you?**
A: A cleanup hook that runs on any exit — normal, error, or signal. Use it to remove temp files / release locks instead of duplicating cleanup in every error path.

**Q: Why is `for f in $(ls *.txt)` buggy?**
A: Word-splits on whitespace — filenames with spaces break. Use `for f in *.txt` (glob, handles spaces) or `find ... -print0 | while read -d ''`.

## Common Pitfalls

- Unquoted variables — `rm $file` on `file="a b"` deletes `a` and `b`.
- Comparing numbers with `==` instead of `-eq` inside `[[ ]]` (works but confusing).
- Forgetting `local` in functions — globals get clobbered silently.
- Parsing `ls` output (filenames with spaces, newlines, glob chars break it).
- Catching `$?` after `if`/`||` — those commands reset `$?`. Capture immediately.

## Related

- [01-linux-essentials.md](01-linux-essentials.md)
- [11-github-actions.md](11-github-actions.md)
- [20-incident-response.md](20-incident-response.md)
