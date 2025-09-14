# Chapter 6: File I/O and Directory Operations

> "Files are the nouns of system administration. Everything else is just verbs." - Unknown Unix Philosopher

If text processing is Perl's superpower, then file handling is its trusty sidekick. Together, they form an unstoppable duo for system administration. This chapter covers everything from basic file operations to advanced techniques like file locking, memory mapping, and atomic operations. By the end, you'll understand why Perl remains the go-to language for file wrangling.

## Opening Files: The Modern Way

### The Three-Argument Open

Forget what you learned in 1999. Modern Perl uses three-argument open:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Old way (DON'T DO THIS)
open FILE, "data.txt";         # Ambiguous and unsafe
open FILE, ">data.txt";        # Easy to accidentally clobber
open FILE, "cat data.txt|";    # Security nightmare!

# Modern way (DO THIS)
open my $fh, '<', 'data.txt' or die "Can't open data.txt: $!";
open my $out, '>', 'output.txt' or die "Can't create output.txt: $!";
open my $append, '>>', 'log.txt' or die "Can't append to log.txt: $!";

# Lexical filehandles = automatic cleanup
{
    open my $temp, '>', 'temp.txt' or die $!;
    print $temp "This file will be closed automatically\n";
}  # $temp goes out of scope, file closed

# Read/write mode
open my $rw, '+<', 'data.txt' or die $!;  # Read and write
open my $trunc, '+>', 'new.txt' or die $!; # Truncate and read/write
```

### Opening Files Safely

```perl
use autodie;  # Automatic error handling

# No need for 'or die' with autodie
open my $fh, '<', 'data.txt';

# But sometimes you want to handle errors yourself
use Try::Tiny;

try {
    open my $fh, '<', 'maybe_exists.txt';
    # Process file
} catch {
    warn "File doesn't exist, using defaults";
    # Use default values
};

# Check if file is readable before opening
if (-r 'data.txt') {
    open my $fh, '<', 'data.txt';
    # Process
}
```

## Reading Files: Choose Your Weapon

### Line by Line (Memory Efficient)

```perl
# The classic way
open my $fh, '<', 'huge_file.txt' or die $!;
while (my $line = <$fh>) {
    chomp $line;
    process_line($line);
}
close $fh;

# With automatic line ending handling
open my $fh, '<:crlf', 'windows_file.txt' or die $!;  # Handle \r\n
while (<$fh>) {
    chomp;  # Works on $_
    say "Line: $_";
}

# Reading with a specific input separator
{
    local $/ = "\n\n";  # Paragraph mode
    open my $fh, '<', 'paragraphs.txt' or die $!;
    while (my $paragraph = <$fh>) {
        process_paragraph($paragraph);
    }
}

# Reading fixed-length records
{
    local $/ = \1024;  # Read 1024 bytes at a time
    open my $fh, '<:raw', 'binary.dat' or die $!;
    while (my $chunk = <$fh>) {
        process_chunk($chunk);
    }
}
```

### Slurping (Read Entire File)

```perl
# Simple slurp
my $content = do {
    open my $fh, '<', 'file.txt' or die $!;
    local $/;  # Slurp mode
    <$fh>;
};

# Slurp to array (one line per element)
open my $fh, '<', 'file.txt' or die $!;
my @lines = <$fh>;
chomp @lines;

# Using Path::Tiny (the modern way)
use Path::Tiny;
my $content = path('file.txt')->slurp_utf8;
my @lines = path('file.txt')->lines({ chomp => 1 });

# Slurp with size check (prevent memory issues)
sub safe_slurp {
    my ($filename, $max_size) = @_;
    $max_size //= 10 * 1024 * 1024;  # 10MB default

    my $size = -s $filename;
    die "File too large ($size bytes)" if $size > $max_size;

    open my $fh, '<', $filename or die $!;
    local $/;
    return <$fh>;
}
```

## Writing Files: Getting Data Out

### Basic Writing

```perl
open my $out, '>', 'output.txt' or die $!;
print $out "Hello, World!\n";
printf $out "Number: %d, String: %s\n", 42, "Perl";
say $out "Automatic newline";  # say adds \n
close $out;

# Write array to file
my @data = qw(apple banana cherry);
open my $fh, '>', 'fruits.txt' or die $!;
say $fh $_ for @data;  # One per line
close $fh;

# Write with specific line endings
open my $fh, '>:crlf', 'windows.txt' or die $!;  # Force \r\n
print $fh "Windows line ending\n";
```

### Atomic Writes (The Safe Way)

```perl
# Never corrupt files with partial writes
sub atomic_write {
    my ($filename, $content) = @_;

    my $temp = "$filename.tmp.$$";  # PID for uniqueness

    open my $fh, '>', $temp or die "Can't write to $temp: $!";
    print $fh $content;
    close $fh or die "Can't close $temp: $!";

    rename $temp, $filename or die "Can't rename $temp to $filename: $!";
}

# Using File::Temp for truly safe temp files
use File::Temp qw(tempfile);

my ($fh, $tempname) = tempfile(DIR => '/tmp');
print $fh "Temporary content\n";
close $fh;
rename $tempname, 'final.txt' or die $!;
```

## File Tests: Know Your Files

Perl's file test operators are legendary:

```perl
my $file = 'test.txt';

# Existence and type
say "Exists" if -e $file;
say "Regular file" if -f $file;
say "Directory" if -d $file;
say "Symbolic link" if -l $file;
say "Named pipe" if -p $file;
say "Socket" if -S $file;
say "Block device" if -b $file;
say "Character device" if -c $file;

# Permissions
say "Readable" if -r $file;
say "Writable" if -w $file;
say "Executable" if -x $file;
say "Owned by me" if -o $file;

# Size and age
my $size = -s $file;  # Size in bytes
my $age_days = -M $file;  # Days since modification
my $access_days = -A $file;  # Days since last access
my $inode_days = -C $file;  # Days since inode change

# Stacking file tests (5.10+)
if (-f -r -w $file) {
    say "Regular file, readable and writable";
}

# The special _ filehandle (cached stat)
if (-e $file) {
    my $size = -s _;  # Uses cached stat from -e test
    my $mtime = -M _;  # Still using cached stat
    say "File is $size bytes, modified $mtime days ago";
}
```

## Directory Operations

### Reading Directories

```perl
# Old school opendir/readdir
opendir my $dh, '.' or die "Can't open directory: $!";
my @files = readdir $dh;
closedir $dh;

# Filter hidden files
my @visible = grep { !/^\./ } @files;

# Get full paths
my $dir = '/etc';
opendir my $dh, $dir or die $!;
my @paths = map { "$dir/$_" } grep { !/^\.\.?$/ } readdir $dh;

# Using glob (shell-style patterns)
my @perl_files = glob("*.pl");
my @all_files = glob("*");
my @hidden = glob(".*");
my @recursive = glob("**/*.txt");  # Requires bsd_glob

# Modern way with Path::Tiny
use Path::Tiny;
my @files = path('.')->children;
my @perl_files = path('.')->children(qr/\.pl$/);

# Recursive directory traversal
use File::Find;
find(sub {
    return unless -f;  # Only files
    return unless /\.log$/;  # Only .log files

    my $path = $File::Find::name;
    my $size = -s;
    say "$path: $size bytes";
}, '/var/log');
```

### Creating and Removing Directories

```perl
# Create directory
mkdir 'new_dir' or die "Can't create directory: $!";
mkdir 'deep/nested/dir';  # Fails if parent doesn't exist

# Create with permissions
mkdir 'secure_dir', 0700 or die $!;  # Owner only

# Create nested directories
use File::Path qw(make_path remove_tree);
make_path('deep/nested/structure') or die "Can't create path: $!";
make_path('dir1', 'dir2', 'dir3', { mode => 0755 });

# Remove directories
rmdir 'empty_dir' or die $!;  # Only works if empty
remove_tree('dir_with_contents');  # Recursive removal
remove_tree('dangerous_dir', { safe => 1 });  # Safe mode

# Using Path::Tiny
use Path::Tiny;
path('some/deep/directory')->mkpath;
path('to_delete')->remove_tree;
```

## Advanced File Operations

### File Locking

```perl
use Fcntl qw(:flock);

open my $fh, '+<', 'shared.dat' or die $!;

# Exclusive lock (writing)
flock($fh, LOCK_EX) or die "Can't lock file: $!";
# ... modify file ...
flock($fh, LOCK_UN);  # Explicit unlock (optional)

# Shared lock (reading)
flock($fh, LOCK_SH) or die "Can't get shared lock: $!";
# ... read file ...

# Non-blocking lock attempt
if (flock($fh, LOCK_EX | LOCK_NB)) {
    # Got the lock
} else {
    warn "File is locked by another process";
}

# Lock with timeout
sub lock_with_timeout {
    my ($fh, $timeout) = @_;

    my $tries = 0;
    until (flock($fh, LOCK_EX | LOCK_NB)) {
        return 0 if ++$tries > $timeout;
        sleep 1;
    }
    return 1;
}
```

### Memory-Mapped Files

```perl
use File::Map qw(map_file);

# Map file to memory
map_file my $map, 'large_file.dat';

# Now $map is like a string, but backed by the file
if ($map =~ /pattern/) {
    say "Found pattern";
}

# Modifications write through to disk
$map =~ s/old/new/g;  # Changes the actual file!

# Explicitly sync to disk
use File::Map qw(sync);
sync($map);
```

### Binary File Handling

```perl
# Reading binary files
open my $fh, '<:raw', 'image.jpg' or die $!;
binmode $fh;  # Alternative way to set binary mode

my $header;
read($fh, $header, 10);  # Read exactly 10 bytes

# Unpack binary data
my ($magic, $width, $height) = unpack('A4 N N', $header);

# Seek to specific position
seek($fh, 1024, 0);  # Absolute position
seek($fh, -10, 2);   # 10 bytes from end
my $pos = tell($fh); # Current position

# Writing binary data
open my $out, '>:raw', 'output.bin' or die $!;
my $packed = pack('N*', 1, 2, 3, 4, 5);
print $out $packed;
```

## Real-World File Processing

### Log Rotation Script

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use File::Copy;
use File::Path qw(make_path);
use POSIX qw(strftime);

sub rotate_logs {
    my ($log_file, $keep_days) = @_;
    $keep_days //= 30;

    return unless -e $log_file;

    # Create archive directory
    my $archive_dir = 'log_archive';
    make_path($archive_dir) unless -d $archive_dir;

    # Generate archive name with timestamp
    my $timestamp = strftime('%Y%m%d_%H%M%S', localtime);
    my $archive_name = "$archive_dir/" .
                      basename($log_file) .
                      ".$timestamp";

    # Compress if large
    if (-s $log_file > 1024 * 1024) {  # > 1MB
        $archive_name .= '.gz';
        system("gzip -c $log_file > $archive_name") == 0
            or die "Compression failed: $?";
        unlink $log_file;
    } else {
        move($log_file, $archive_name) or die "Move failed: $!";
    }

    # Create new empty log file
    open my $fh, '>', $log_file or die $!;
    close $fh;

    # Clean old archives
    clean_old_archives($archive_dir, $keep_days);
}

sub clean_old_archives {
    my ($dir, $keep_days) = @_;

    opendir my $dh, $dir or die $!;
    while (my $file = readdir $dh) {
        next if $file =~ /^\./;
        my $path = "$dir/$file";
        next unless -f $path;

        if (-M $path > $keep_days) {
            unlink $path or warn "Can't delete $path: $!";
            say "Deleted old archive: $file";
        }
    }
    closedir $dh;
}

# Use it
rotate_logs('/var/log/myapp.log', 30);
```

### File Synchronization

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use File::Find;
use File::Copy;
use Digest::MD5;
use Path::Tiny;

sub sync_directories {
    my ($source, $dest) = @_;

    die "Source doesn't exist: $source" unless -d $source;
    make_path($dest) unless -d $dest;

    my %source_files;

    # Scan source directory
    find(sub {
        return unless -f;
        my $rel_path = $File::Find::name;
        $rel_path =~ s/^\Q$source\E\/?//;

        $source_files{$rel_path} = {
            size => -s $_,
            mtime => -M $_,
            md5 => calculate_md5($_),
        };
    }, $source);

    # Sync files
    for my $file (keys %source_files) {
        my $src_path = "$source/$file";
        my $dst_path = "$dest/$file";

        if (!-e $dst_path ||
            files_differ($src_path, $dst_path)) {

            # Create directory structure
            my $dst_dir = path($dst_path)->parent;
            $dst_dir->mkpath unless -d $dst_dir;

            # Copy file
            copy($src_path, $dst_path)
                or die "Copy failed: $!";
            say "Synced: $file";
        }
    }

    # Remove files not in source
    find(sub {
        return unless -f;
        my $rel_path = $File::Find::name;
        $rel_path =~ s/^\Q$dest\E\/?//;

        unless (exists $source_files{$rel_path}) {
            unlink $File::Find::name;
            say "Removed: $rel_path";
        }
    }, $dest);
}

sub calculate_md5 {
    my ($file) = @_;
    open my $fh, '<:raw', $file or die $!;
    my $md5 = Digest::MD5->new;
    $md5->addfile($fh);
    return $md5->hexdigest;
}

sub files_differ {
    my ($file1, $file2) = @_;

    # Quick checks first
    return 1 if -s $file1 != -s $file2;

    # Then MD5
    return calculate_md5($file1) ne calculate_md5($file2);
}
```

### CSV to JSON Converter

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Text::CSV;
use JSON::XS;

sub csv_to_json {
    my ($csv_file, $json_file) = @_;

    my $csv = Text::CSV->new({ binary => 1, auto_diag => 1 });
    open my $fh, '<:encoding(utf8)', $csv_file or die $!;

    # Read header
    my $headers = $csv->getline($fh);
    $csv->column_names(@$headers);

    # Read all rows as hashrefs
    my @data;
    while (my $row = $csv->getline_hr($fh)) {
        push @data, $row;
    }

    close $fh;

    # Write JSON
    my $json = JSON::XS->new->utf8->pretty->canonical;
    open my $out, '>:encoding(utf8)', $json_file or die $!;
    print $out $json->encode(\@data);
    close $out;

    say "Converted " . scalar(@data) . " records";
}

# Usage
csv_to_json('data.csv', 'data.json');
```

## File I/O Best Practices

1. **Always use lexical filehandles** - They auto-close and are scoped
2. **Use three-argument open** - Safer and clearer
3. **Check return values** - Or use autodie
4. **Consider memory usage** - Don't slurp huge files
5. **Use Path::Tiny for modern code** - It handles many edge cases
6. **Lock files when needed** - Prevent corruption in concurrent access
7. **Make writes atomic** - Write to temp file, then rename
8. **Handle encodings explicitly** - Use :encoding(UTF-8) layer
9. **Clean up temp files** - Use File::Temp or END blocks
10. **Validate file paths** - Never trust user input for file names

## Common Gotchas

### The Newline Problem

```perl
# Problem: Forgetting chomp
while (<$fh>) {
    push @lines, $_;  # Includes newlines!
}

# Solution
while (<$fh>) {
    chomp;
    push @lines, $_;
}
```

### The Encoding Trap

```perl
# Problem: Mojibake (garbled characters)
open my $fh, '<', 'utf8_file.txt';

# Solution: Specify encoding
open my $fh, '<:encoding(UTF-8)', 'utf8_file.txt';
```

### The Buffering Issue

```perl
# Problem: Output doesn't appear immediately
print $log_fh "Important message";
# Program crashes, message never written!

# Solution: Disable buffering
$log_fh->autoflush(1);
# Or
select($log_fh); $| = 1; select(STDOUT);
```

## Wrapping Up

File I/O in Perl is both powerful and pragmatic. The language gives you high-level conveniences (slurping, Path::Tiny) and low-level control (seek, sysread) in equal measure. This flexibility is why Perl remains indispensable for system administration tasks.

Remember: files are just streams of bytes. Perl gives you dozens of ways to read, write, and manipulate those bytes. Choose the right tool for your specific task, and always consider edge cases like file locking, encoding, and error handling.

---

*Next up: Advanced text processing. We'll go beyond basic regex to explore Perl's sophisticated text manipulation capabilities, including parsing structured formats, template processing, and building your own mini-languages.*