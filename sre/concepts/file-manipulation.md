# File Manipulation — Linux CLI

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/concepts/log-analysis]], [[sre/concepts/disk-and-io]]

## find — locate files

```bash
# Files modified in last 24 hours
find /var/log -mtime -1 -type f

# Files larger than 100MB
find / -type f -size +100M 2>/dev/null

# Files with specific permissions (world-writable — security audit)
find / -perm -o+w -type f 2>/dev/null

# Find and delete temp files older than 7 days
find /tmp -type f -mtime +7 -delete

# Find and execute: gzip all .log files
find /var/log -name "*.log" -exec gzip {} \;

# Find by name, case-insensitive
find /home -iname "*.py" -type f

# Find files owned by a user
find /home -user deploy -type f
```

**Senior gotcha**: `find / -name foo` will error on unreadable dirs — redirect `2>/dev/null` to suppress. In scripts, use `find ... -print0 | xargs -0` to safely handle filenames with spaces.

---

## grep — search content

```bash
# Basic search
grep "ERROR" app.log

# Case-insensitive, recursive, show line numbers
grep -rin "out of memory" /var/log/

# Show 3 lines of context around each match
grep -C 3 "FATAL" app.log

# Invert match (lines NOT matching)
grep -v "DEBUG" app.log

# Count matching lines
grep -c "ERROR" app.log

# Match whole word only
grep -w "fail" app.log

# Extended regex (alternation, +, ?)
grep -E "ERROR|FATAL|CRITICAL" app.log

# Print only the matching part (not the whole line)
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" access.log   # extract IPs

# Search multiple files, show filename
grep -l "OutOfMemoryError" /var/log/*.log

# Grep within gzipped files
zgrep "ERROR" /var/log/app.log.gz
```

---

## awk — structured text processing

awk processes input line-by-line, splitting each into fields (`$1`, `$2`, ..., `$NF`). Default delimiter is whitespace; use `-F` to set a custom one.

```bash
# Print specific columns from a log
awk '{print $1, $4}' access.log           # IP and timestamp

# Custom delimiter (CSV/TSV)
awk -F',' '{print $2}' data.csv           # second column

# Filter lines where field > threshold
awk '$5 > 500' access.log                 # requests where status code field > 500

# Sum a column
awk '{sum += $7} END {print sum}' access.log   # total bytes served

# Count occurrences of a field value
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log

# Print lines between two patterns
awk '/START/,/END/' logfile

# If/else logic
awk '{if ($9 >= 500) print "ERROR:", $0; else print "OK:", $0}' access.log
```

**Classic SRE one-liner — top 10 IPs by request count:**
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

---

## sed — stream editing

```bash
# Substitute first occurrence per line
sed 's/foo/bar/' file.txt

# Substitute all occurrences (global flag)
sed 's/foo/bar/g' file.txt

# In-place edit (macOS needs -i '')
sed -i '' 's/old_password/new_password/g' config.txt
sed -i 's/old_password/new_password/g' config.txt   # Linux

# Delete lines matching a pattern
sed '/^#/d' config.txt          # remove comment lines
sed '/^$/d' config.txt          # remove blank lines

# Print only lines 10–20
sed -n '10,20p' file.txt

# Insert a line after a match
sed '/\[section\]/a new_key=value' config.ini

# Delete a line containing a pattern
sed '/deprecated_setting/d' config.ini

# Replace newlines (tricky — use tr instead)
tr '\n' ',' < file.txt
```

---

## sort, uniq, cut — data pipeline building blocks

```bash
# Sort numerically, descending
sort -rn file.txt

# Sort by second field (delimiter: colon)
sort -t':' -k2 /etc/passwd

# Sort by file size (human-readable output from du)
du -sh * | sort -h

# Count and sort unique lines
sort file.txt | uniq -c | sort -rn

# Only show duplicate lines
sort file.txt | uniq -d

# Cut first and third columns (colon-delimited)
cut -d':' -f1,3 /etc/passwd

# Extract character range
cut -c1-8 file.txt
```

---

## xargs — build and execute pipelines

xargs is the glue between commands that output lists and commands that don't accept piped input.

```bash
# Delete all .tmp files found by find
find /tmp -name "*.tmp" | xargs rm -f

# Safely handle filenames with spaces (-0 pairs with find -print0)
find . -name "*.log" -print0 | xargs -0 gzip

# Run 4 parallel jobs
find . -name "*.py" -print0 | xargs -0 -P 4 python lint.py

# Limit to N arguments per invocation
cat urls.txt | xargs -n 1 curl -O

# Confirm each command before running
find . -name "*.bak" | xargs -I{} -p rm {}
```

---

## File permissions

```bash
# chmod symbolic
chmod u+x script.sh         # owner: add execute
chmod go-w file.txt         # group+other: remove write
chmod 755 dir/              # rwxr-xr-x

# chmod numeric: owner=7(rwx), group=5(r-x), other=4(r--)
chmod 754 file.sh

# Change ownership
chown user:group file.txt
chown -R deploy:deploy /var/app/

# umask: default permission mask
umask 022   # new files get 644, new dirs get 755
umask 027   # new files get 640, new dirs get 750

# setuid/setgid/sticky bit
chmod u+s /usr/bin/passwd   # setuid: runs as file owner
chmod +t /tmp               # sticky: only owner can delete
```

**Security interview angle**: "Find world-writable files" → `find / -perm -o+w -type f`; "Find setuid binaries" → `find / -perm -u+s -type f`

---

## File descriptors and /proc

```bash
# List all file descriptors for a process (by PID)
lsof -p 1234

# How many FDs is process 1234 using?
ls /proc/1234/fd | wc -l

# Find process holding a deleted-but-open file (why disk is full after deletion)
lsof | grep deleted

# Check FD limits
ulimit -n                        # current process soft limit
cat /proc/sys/fs/file-max        # system-wide max
```

**Classic SRE scenario**: you deleted a large log file but `df` still shows no free space. The file is still open by the logging process — `lsof | grep deleted` finds it; restart the process or send `SIGHUP` to reopen the log.

---

## Pipes, redirection, and process substitution

```bash
# Pipe: stdout of left → stdin of right
cat access.log | grep ERROR | awk '{print $1}'

# Redirect stdout to file (overwrite / append)
command > out.txt
command >> out.txt

# Redirect stderr
command 2> err.txt
command > out.txt 2>&1       # stdout + stderr to same file
command &> out.txt           # bash shorthand for the above

# Discard output
command > /dev/null 2>&1

# Process substitution (treats command output as a file)
diff <(sort file1.txt) <(sort file2.txt)

# Here-string
grep "pattern" <<< "this is my string"

# tee: write to both stdout and file (useful in pipelines)
tail -f app.log | tee -a filtered.log | grep ERROR
```

---

## Common interview angles
- "Parse an Apache/nginx access log and find the top 10 slowest endpoints" → `awk` on response-time field + `sort -rn` + `head`
- "Find all files owned by a deleted user (UID 1234)" → `find / -uid 1234`
- "In-place replace a config value across 50 files" → `grep -rl 'old' . | xargs sed -i 's/old/new/g'`
- "Why is disk full even after I deleted the log file?" → `lsof | grep deleted`
- "How do you tail multiple log files at once?" → `tail -f file1.log file2.log` or `multitail`

## Sources
*(none yet)*
