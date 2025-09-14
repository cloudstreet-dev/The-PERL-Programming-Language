# Appendix A: Perl One-Liners Cookbook

*"Sometimes the most powerful programs are the ones that fit on a single line."*

Perl's conciseness and expressiveness shine brightest in one-liners—those magical incantations that transform data, search files, and automate tasks, all from the command line. This cookbook contains battle-tested one-liners for real-world system administration tasks.

## Text Processing

### Search and Replace

```bash
# Replace text in a file (in-place)
perl -pi -e 's/old_text/new_text/g' file.txt

# Replace with backup
perl -pi.bak -e 's/old_text/new_text/g' file.txt

# Replace only on lines matching a pattern
perl -pi -e 's/old/new/g if /pattern/' file.txt

# Replace across multiple files
perl -pi -e 's/old/new/g' *.txt

# Case-insensitive replace
perl -pi -e 's/old/new/gi' file.txt

# Replace with captured groups
perl -pi -e 's/(\w+)\.old\.com/$1.new.com/g' file.txt

# Replace newlines with commas
perl -pe 's/\n/,/g' file.txt

# Remove trailing whitespace
perl -pi -e 's/\s+$//' file.txt

# Convert DOS to Unix line endings
perl -pi -e 's/\r\n/\n/g' file.txt

# Convert Unix to DOS line endings
perl -pi -e 's/\n/\r\n/g' file.txt

# Replace multiple spaces with single space
perl -pi -e 's/ +/ /g' file.txt

# Remove blank lines
perl -ni -e 'print unless /^$/' file.txt

# Replace tabs with spaces
perl -pi -e 's/\t/    /g' file.txt

# URL encode
perl -MURI::Escape -e 'print uri_escape($ARGV[0])' "hello world"

# URL decode
perl -MURI::Escape -e 'print uri_unescape($ARGV[0])' "hello%20world"

# HTML encode
perl -MHTML::Entities -e 'print encode_entities($ARGV[0])' "<h1>Title</h1>"

# Base64 encode
perl -MMIME::Base64 -e 'print encode_base64("text")'

# Base64 decode
perl -MMIME::Base64 -e 'print decode_base64("dGV4dA==")'
```

### Line Operations

```bash
# Print specific line number
perl -ne 'print if $. == 42' file.txt

# Print lines 10-20
perl -ne 'print if 10..20' file.txt

# Print every nth line
perl -ne 'print if $. % 3 == 0' file.txt

# Number all lines
perl -pe '$_ = "$. $_"' file.txt

# Print line before and after match
perl -ne 'print $prev if /pattern/; $prev = $_' file.txt

# Remove duplicate lines
perl -ne 'print unless $seen{$_}++' file.txt

# Print lines longer than 80 characters
perl -ne 'print if length > 80' file.txt

# Print shortest line
perl -ne '$min = $_ if !defined $min || length($_) < length($min); END {print $min}' file.txt

# Print longest line
perl -ne '$max = $_ if length($_) > length($max // ""); END {print $max}' file.txt

# Reverse lines (tac)
perl -e 'print reverse <>' file.txt

# Shuffle lines randomly
perl -MList::Util=shuffle -e 'print shuffle <>' file.txt

# Sort lines
perl -e 'print sort <>' file.txt

# Sort numerically
perl -e 'print sort {$a <=> $b} <>' file.txt

# Sort by column
perl -e 'print sort {(split /\s+/, $a)[2] cmp (split /\s+/, $b)[2]} <>' file.txt
```

### Field and Column Operations

```bash
# Print first field (like cut -f1)
perl -lane 'print $F[0]' file.txt

# Print specific columns
perl -lane 'print "@F[0,2,4]"' file.txt

# Print last field
perl -lane 'print $F[-1]' file.txt

# Sum numbers in first column
perl -lane '$sum += $F[0]; END {print $sum}' file.txt

# Average of column
perl -lane '$sum += $F[0]; $count++; END {print $sum/$count}' file.txt

# Join fields with comma
perl -lane 'print join(",", @F)' file.txt

# Print fields in reverse order
perl -lane 'print join(" ", reverse @F)' file.txt

# Count fields per line
perl -lane 'print scalar @F' file.txt

# Print lines with exactly n fields
perl -lane 'print if @F == 5' file.txt

# Transpose rows and columns
perl -lane 'push @{$c[$_]}, $F[$_] for 0..$#F; END {print "@$_" for @c}' file.txt

# Convert CSV to TSV
perl -pe 's/,/\t/g' file.csv

# Parse CSV properly
perl -MText::CSV -e 'my $csv = Text::CSV->new(); while (<>) {$csv->parse($_); print join("\t", $csv->fields), "\n"}' file.csv
```

## File Operations

### File Management

```bash
# Find files modified in last 24 hours
perl -e 'print "$_\n" for grep {-M $_ < 1} glob("*")'

# Find files larger than 1MB
perl -e 'print "$_\n" for grep {-s $_ > 1_000_000} glob("*")'

# Rename files (add prefix)
perl -e 'rename $_, "prefix_$_" for glob("*.txt")'

# Rename files (change extension)
perl -e 'for (glob("*.txt")) {$new = $_; $new =~ s/\.txt$/.bak/; rename $_, $new}'

# Delete empty files
perl -e 'unlink for grep {-z $_} glob("*")'

# Create backup copies
perl -e 'for (glob("*.conf")) {system("cp", $_, "$_.backup")}'

# List directories only
perl -e 'print "$_\n" for grep {-d $_} glob("*")'

# List files recursively
perl -MFile::Find -e 'find(sub {print "$File::Find::name\n" if -f}, ".")'

# Calculate total size of files
perl -e '$total += -s $_ for glob("*"); print "$total\n"'

# Find duplicate files by size
perl -e 'for (glob("*")) {push @{$files{-s $_}}, $_} for (values %files) {print "@$_\n" if @$_ > 1}'

# Touch files (update timestamp)
perl -e 'utime(undef, undef, $_) for glob("*.txt")'

# Make files executable
perl -e 'chmod 0755, $_ for glob("*.pl")'

# Find broken symlinks
perl -e 'print "$_\n" for grep {-l $_ && !-e $_} glob("*")'
```

### Content Analysis

```bash
# Count lines in file
perl -ne 'END {print $.}' file.txt

# Count words
perl -ne '$w += split; END {print "$w\n"}' file.txt

# Count characters
perl -ne '$c += length; END {print "$c\n"}' file.txt

# Frequency count of words
perl -ne 'for (split) {$freq{lc $_}++} END {print "$_: $freq{$_}\n" for sort {$freq{$b} <=> $freq{$a}} keys %freq}' file.txt

# Find longest word
perl -ne 'for (split) {$max = $_ if length($_) > length($max)} END {print "$max\n"}' file.txt

# Extract email addresses
perl -ne 'print "$1\n" while /([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})/g' file.txt

# Extract URLs
perl -ne 'print "$1\n" while m!(https?://[^\s]+)!g' file.txt

# Extract IP addresses
perl -ne 'print "$1\n" while /(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})/g' file.txt

# Extract phone numbers (US format)
perl -ne 'print "$1\n" while /(\d{3}[-.]?\d{3}[-.]?\d{4})/g' file.txt

# Check for binary files
perl -e 'for (glob("*")) {print "$_\n" if -B $_}'

# Find files containing text
perl -e 'for $f (glob("*.txt")) {open F, $f; print "$f\n" if grep /pattern/, <F>; close F}'
```

## System Administration

### Process Management

```bash
# Kill processes by name
perl -e 'kill "TERM", grep {/processname/} map {/^\s*(\d+)/; $1} `ps aux`'

# Show process tree
perl -e 'for (`ps aux`) {/^(\S+)\s+(\d+)\s+\S+\s+\S+\s+\S+\s+\S+\s+\S+\s+\S+\s+(.*)/ && print "$2 $3\n"}'

# Monitor CPU usage
perl -e 'while (1) {$cpu = `top -bn1 | grep "Cpu(s)"`; print $cpu; sleep 5}'

# Show memory usage
perl -ne 'print if /^Mem/' < /proc/meminfo

# List open ports
perl -ne 'print if /:[\dA-F]{4}\s+0{8}:0{4}/' < /proc/net/tcp

# Show disk usage over threshold
perl -e 'for (`df -h`) {print if /(\d+)%/ && $1 > 80}'

# Parse system logs
perl -ne 'print if /ERROR|WARNING/' /var/log/syslog

# Count log entries by hour
perl -ne 'if (/(\d{2}):\d{2}:\d{2}/) {$hours{$1}++} END {print "$_: $hours{$_}\n" for sort keys %hours}' /var/log/messages
```

### Network Operations

```bash
# Ping sweep
perl -e 'for (1..254) {system("ping -c 1 -W 1 192.168.1.$_ &")}'

# Extract MAC addresses
perl -ne 'print "$1\n" while /((?:[0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2})/g' file.txt

# Parse Apache logs
perl -ne '/^(\S+).*\[([^\]]+)\]\s+"([^"]+)"\s+(\d+)/ && print "$1 $2 $3 $4\n"' access.log

# Count 404 errors
perl -ne '$c++ if / 404 /; END {print "$c\n"}' access.log

# Top IP addresses in log
perl -ne '/^(\S+)/ && $ip{$1}++; END {print "$_: $ip{$_}\n" for (sort {$ip{$b} <=> $ip{$a}} keys %ip)[0..9]}' access.log

# Monitor bandwidth (rough)
perl -e 'while (1) {$rx1 = `cat /sys/class/net/eth0/statistics/rx_bytes`; sleep 1; $rx2 = `cat /sys/class/net/eth0/statistics/rx_bytes`; printf "%.2f KB/s\n", ($rx2-$rx1)/1024}'

# Parse netstat output
perl -ne 'print "$1:$2\n" if /tcp\s+\d+\s+\d+\s+([\d.]+):(\d+)/' < <(netstat -an)

# DNS lookup
perl -MSocket -e 'print inet_ntoa(inet_aton($ARGV[0]))' hostname
```

## Data Processing

### JSON Operations

```bash
# Pretty print JSON
perl -MJSON::PP -e 'print JSON::PP->new->pretty->encode(decode_json(<>))' file.json

# Extract field from JSON
perl -MJSON::PP -ne 'print decode_json($_)->{field}, "\n"' file.json

# Convert JSON to CSV
perl -MJSON::PP -ne '$j = decode_json($_); print join(",", @{$j}{qw(field1 field2 field3)}), "\n"' file.json

# Filter JSON array
perl -MJSON::PP -e '$j = decode_json(join "", <>); print encode_json([grep {$_->{age} > 18} @$j])' file.json

# Merge JSON files
perl -MJSON::PP -e '@all = map {decode_json(do {local $/; open F, $_; <F>})} @ARGV; print encode_json(\@all)' *.json
```

### XML Processing

```bash
# Extract XML tags
perl -ne 'print "$1\n" while /<tag>([^<]+)<\/tag>/g' file.xml

# Convert XML to text
perl -pe 's/<[^>]*>//g' file.xml

# Pretty print XML
perl -MXML::Tidy -e 'my $tidy = XML::Tidy->new(filename => $ARGV[0]); $tidy->tidy(); print $tidy->toString()' file.xml

# Extract attributes
perl -ne 'print "$1\n" while /attribute="([^"]+)"/g' file.xml
```

### Database Operations

```bash
# Quick SQLite query
perl -MDBI -e '$dbh = DBI->connect("dbi:SQLite:dbname=test.db"); print "@$_\n" for @{$dbh->selectall_arrayref("SELECT * FROM users")}'

# Export to CSV
perl -MDBI -e '$dbh = DBI->connect("dbi:SQLite:dbname=test.db"); $sth = $dbh->prepare("SELECT * FROM users"); $sth->execute(); while ($row = $sth->fetchrow_arrayref) {print join(",", @$row), "\n"}'

# Import from CSV
perl -MDBI -MText::CSV -e '$csv = Text::CSV->new(); $dbh = DBI->connect("dbi:SQLite:dbname=test.db"); while (<>) {$csv->parse($_); $dbh->do("INSERT INTO users VALUES (?, ?, ?)", undef, $csv->fields)}'
```

## Date and Time

```bash
# Current timestamp
perl -e 'print time, "\n"'

# Current date
perl -e 'print scalar localtime, "\n"'

# Format date
perl -MPOSIX -e 'print strftime("%Y-%m-%d %H:%M:%S\n", localtime)'

# Days between dates
perl -MTime::Piece -e '$t1 = Time::Piece->strptime("2024-01-01", "%Y-%m-%d"); $t2 = Time::Piece->strptime("2024-12-31", "%Y-%m-%d"); print int(($t2 - $t1) / 86400), "\n"'

# Convert epoch to date
perl -e 'print scalar localtime(1704067200), "\n"'

# Add days to date
perl -MTime::Piece -MTime::Seconds -e '$t = localtime; $t += ONE_DAY * 7; print $t->strftime("%Y-%m-%d\n")'

# File age in days
perl -e 'print int(-M $ARGV[0]), "\n"' file.txt

# Show files modified today
perl -e 'print "$_\n" for grep {int(-M $_) == 0} glob("*")'
```

## Mathematical Operations

```bash
# Calculator
perl -e 'print eval($ARGV[0]), "\n"' "2 + 2 * 3"

# Sum numbers from stdin
perl -ne '$sum += $_; END {print "$sum\n"}'

# Generate random numbers
perl -e 'print int(rand(100)), "\n" for 1..10'

# Factorial
perl -e '$f = 1; $f *= $_ for 1..$ARGV[0]; print "$f\n"' 5

# Prime numbers
perl -e 'for $n (2..100) {$p = 1; $n % $_ or $p = 0 for 2..sqrt($n); print "$n " if $p}'

# Fibonacci sequence
perl -e '$a = $b = 1; print "$a "; for (1..20) {print "$b "; ($a, $b) = ($b, $a + $b)}'

# Convert hex to decimal
perl -e 'print hex($ARGV[0]), "\n"' FF

# Convert decimal to hex
perl -e 'printf "%X\n", $ARGV[0]' 255

# Convert binary to decimal
perl -e 'print oct("0b" . $ARGV[0]), "\n"' 11111111

# Statistics
perl -ne 'push @v, $_; END {$sum += $_ for @v; $avg = $sum/@v; $var += ($_-$avg)**2 for @v; print "Mean: $avg, StdDev: ", sqrt($var/@v), "\n"}' numbers.txt
```

## Encoding and Conversion

```bash
# Convert to uppercase
perl -pe '$_ = uc'

# Convert to lowercase
perl -pe '$_ = lc'

# Title case
perl -pe 's/\b(\w)/\u$1/g'

# ROT13 encoding
perl -pe 'tr/A-Za-z/N-ZA-Mn-za-m/'

# Convert ASCII to hex
perl -ne 'print unpack("H*", $_), "\n"'

# Convert hex to ASCII
perl -ne 'print pack("H*", $_)'

# Unicode operations
perl -CS -pe 's/\N{U+00E9}/e/g'  # Replace é with e

# Convert encoding
perl -MEncode -pe '$_ = encode("UTF-8", decode("ISO-8859-1", $_))'
```

## Quick Utilities

```bash
# Generate UUID
perl -MData::UUID -e 'print Data::UUID->new->create_str, "\n"'

# Password generator
perl -e 'print map {("a".."z","A".."Z",0..9)[rand 62]} 1..16; print "\n"'

# Check if port is open
perl -MIO::Socket::INET -e 'print IO::Socket::INET->new("$ARGV[0]:$ARGV[1]") ? "Open\n" : "Closed\n"' hostname 80

# Simple HTTP GET
perl -MLWP::Simple -e 'print get($ARGV[0])' 'http://example.com'

# Send email
perl -MMIME::Lite -e 'MIME::Lite->new(From => "sender@example.com", To => "recipient@example.com", Subject => "Test", Data => "Hello")->send'

# Benchmark code
perl -MBenchmark -e 'timethis(1000000, sub {$x = 1 + 1})'

# Create tar archive
perl -MArchive::Tar -e 'Archive::Tar->create_archive("archive.tar.gz", COMPRESS_GZIP, glob("*.txt"))'

# Weather (requires Internet)
perl -MLWP::Simple -e 'print +(get("http://wttr.in/?format=3") =~ /: (.+)/)[0], "\n"'

# QR code generator
perl -MImager::QRCode -e 'Imager::QRCode->new->plot($ARGV[0])->write(file => "qr.png")' "Hello World"

# Simple web server
perl -MIO::All -e 'io(":8080")->fork->accept->(sub {$_[0] < io("index.html")})'

# Watch file for changes
perl -e 'while (1) {$new = -M $ARGV[0]; print "Changed\n" if $new != $old; $old = $new; sleep 1}' file.txt

# Bulk email validator
perl -MEmail::Valid -ne 'chomp; print "$_\n" if Email::Valid->address($_)' emails.txt

# Simple port scanner
perl -MIO::Socket::INET -e 'for (1..1024) {print "$_\n" if IO::Socket::INET->new("$ARGV[0]:$_")}'  hostname

# Directory tree
perl -MFile::Find -e 'find(sub {$d = $File::Find::dir =~ tr!/!!; print "  " x $d, "$_\n"}, ".")'
```

## Performance One-Liners

```bash
# Profile code execution
perl -d:NYTProf -e 'your_code_here'

# Memory usage
perl -e 'system("ps -o pid,vsz,rss,comm -p $$")'

# Time execution
perl -MTime::HiRes=time -e '$t = time; system($ARGV[0]); printf "%.3f seconds\n", time - $t' "command"

# Count module load time
perl -MTime::HiRes=time -e '$t = time; require Some::Module; printf "%.3f seconds\n", time - $t'
```

## Tips for Writing One-Liners

1. **Essential Options**
   - `-e`: Execute code
   - `-n`: Loop over input lines (while (<>) {...})
   - `-p`: Loop and print (while (<>) {...; print})
   - `-i`: In-place editing
   - `-l`: Auto chomp and add newlines
   - `-a`: Auto-split into @F
   - `-F`: Set field separator for -a

2. **Special Variables**
   - `$_`: Current line/topic
   - `$.`: Line number
   - `@F`: Fields (with -a)
   - `$/`: Input record separator
   - `$\`: Output record separator

3. **Module Loading**
   - `-M`: Load module (-MJSON::PP)
   - `-m`: Load module without importing

4. **Common Patterns**
   - `BEGIN {}`: Run before loop
   - `END {}`: Run after loop
   - `next if`: Skip lines
   - `last if`: Stop processing

Remember: One-liners are powerful but can become unreadable. If a one-liner grows too complex, consider writing a proper script. The goal is efficiency, not obscurity.

*"The best one-liner is the one that gets the job done and is still understandable tomorrow."*