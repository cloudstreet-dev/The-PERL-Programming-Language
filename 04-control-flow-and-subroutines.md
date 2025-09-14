# Chapter 4: Control Flow and Subroutines

> "Do what I mean, not what I say." - The Perl Philosophy

Control flow in Perl is where the language's personality really shines. Yes, you have your standard if/else and loops. But Perl adds statement modifiers, multiple ways to loop, and some of the most flexible subroutine handling you'll find anywhere. This chapter will show you how to write Perl that reads like English while being devastatingly effective.

## Statement Modifiers: Perl's Poetry

Most languages make you write:
```perl
if ($condition) {
    do_something();
}
```

Perl lets you write:
```perl
do_something() if $condition;
```

This isn't just syntactic sugar—it's a different way of thinking about code:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Traditional vs Perl style
my $file = 'config.txt';

# Traditional
if (-e $file) {
    say "File exists";
}

# Perl style - when the action is more important than the condition
say "File exists" if -e $file;

# Works with all control structures
die "Config not found!" unless -e $file;
say $_ while <DATA>;
$count++ for @items;
warn "Retrying..." until connect_to_server();

# Especially elegant for guard clauses
sub process_file {
    my ($filename) = @_;
    return unless defined $filename;
    return unless -e $filename;
    return unless -r $filename;

    # Main logic here, uncluttered by nested ifs
}
```

## The Many Faces of Conditionals

### if, elsif, else, unless

Perl's `unless` is the logical opposite of `if`. Use it when the negative condition is more natural:

```perl
# Awkward double negative
if (!$user->is_authenticated) {
    redirect_to_login();
}

# Clear and direct
unless ($user->is_authenticated) {
    redirect_to_login();
}

# Even clearer as a modifier
redirect_to_login() unless $user->is_authenticated;

# Complex conditionals
my $status = get_server_status();
if ($status eq 'running') {
    say "All systems operational";
} elsif ($status eq 'degraded') {
    warn "Performance degraded";
    notify_ops_team();
} elsif ($status eq 'maintenance') {
    say "Scheduled maintenance in progress";
} else {
    die "Unknown status: $status";
}
```

### The Ternary Operator

For simple conditionals, the ternary operator keeps things concise:

```perl
my $message = $count > 0 ? "$count items found" : "No items found";

# Nested ternaries (use sparingly!)
my $status = $code == 200 ? 'success' :
             $code == 404 ? 'not found' :
             $code == 500 ? 'server error' : 'unknown';

# Often clearer as a dispatch table
my %status_messages = (
    200 => 'success',
    404 => 'not found',
    500 => 'server error',
);
my $status = $status_messages{$code} // 'unknown';
```

### given/when (Experimental but Useful)

Perl's switch statement is more powerful than most:

```perl
use feature 'switch';
no warnings 'experimental::smartmatch';

given ($command) {
    when ('start')   { start_service() }
    when ('stop')    { stop_service() }
    when ('restart') { stop_service(); start_service() }
    when (/^stat/)  { show_status() }  # Regex matching!
    when ([qw(help ? h)]) { show_help() }  # Array matching!
    default { die "Unknown command: $command" }
}
```

## Loops: There's More Than One Way To Iterate

### The C-Style for Loop

Traditional but verbose:

```perl
for (my $i = 0; $i < 10; $i++) {
    say "Count: $i";
}
```

### The Perl-Style foreach

Much more common and readable:

```perl
# foreach and for are synonyms
for my $item (@items) {
    process($item);
}

# Default variable $_
for (@items) {
    process($_);  # Or just: process()
}

# Range operator
for my $num (1..100) {
    say $num if $num % 15 == 0;  # FizzBuzz anyone?
}

# Hash iteration
for my $key (keys %hash) {
    say "$key: $hash{$key}";
}

# Multiple items at once
my @pairs = qw(a 1 b 2 c 3);
for (my $i = 0; $i < @pairs; $i += 2) {
    my ($letter, $number) = @pairs[$i, $i+1];
    say "$letter = $number";
}
```

### while and until

Perfect for conditional iteration:

```perl
# Read file line by line
while (my $line = <$fh>) {
    chomp $line;
    process($line);
}

# With default variable
while (<$fh>) {
    chomp;  # Operates on $_
    process($_);
}

# until is while's opposite
my $attempts = 0;
until (connect_to_database() || $attempts++ > 5) {
    sleep 2 ** $attempts;  # Exponential backoff
}
```

### Loop Control

Perl provides fine-grained loop control:

```perl
# next - skip to next iteration
for my $file (@files) {
    next if $file =~ /^\./;  # Skip hidden files
    process($file);
}

# last - exit loop
for my $line (<$fh>) {
    last if $line =~ /^__END__$/;
    process($line);
}

# redo - restart current iteration
my $tries = 0;
for my $url (@urls) {
    my $response = fetch($url);
    if (!$response && $tries++ < 3) {
        sleep 1;
        redo;  # Try same URL again
    }
    $tries = 0;  # Reset for next URL
}

# Labels for nested loops
FILE: for my $file (@files) {
    LINE: while (my $line = <$file>) {
        next FILE if $line =~ /SKIP_FILE/;
        next LINE if $line =~ /^\s*#/;
        process($line);
    }
}
```

### The Elegant map and grep

Functional programming in Perl:

```perl
# Transform lists with map
my @files = qw(foo.txt bar.log baz.conf);
my @sizes = map { -s $_ } @files;  # Get file sizes
my @upper = map { uc } @words;     # Uppercase all

# Filter lists with grep
my @configs = grep { /\.conf$/ } @files;
my @large_files = grep { -s $_ > 1024 * 1024 } @files;

# Chain them
my @large_logs = grep { -s $_ > 1024 * 1024 }
                 map  { "$logdir/$_" }
                 grep { /\.log$/ }
                 readdir($dh);

# Schwartzian Transform (sorting optimization)
my @sorted = map  { $_->[0] }
            sort { $a->[1] <=> $b->[1] }
            map  { [$_, -s $_] }
            @files;
```

## Subroutines: Functions with Flexibility

### Basic Subroutines

```perl
# Old style (still common)
sub greet {
    my ($name) = @_;  # Unpack @_
    $name //= 'World';
    say "Hello, $name!";
}

# Modern style with signatures (Perl 5.20+)
use feature 'signatures';
no warnings 'experimental::signatures';

sub greet_modern($name = 'World') {
    say "Hello, $name!";
}

# Multiple parameters
sub calculate($x, $y, $operation = '+') {
    return $x + $y if $operation eq '+';
    return $x - $y if $operation eq '-';
    return $x * $y if $operation eq '*';
    return $x / $y if $operation eq '/';
    die "Unknown operation: $operation";
}
```

### Return Values and Context

Subroutines can be context-aware:

```perl
sub get_data {
    my @data = qw(apple banana cherry);

    # wantarray tells us the context
    if (wantarray) {
        return @data;  # List context
    } elsif (defined wantarray) {
        return scalar @data;  # Scalar context
    } else {
        say "@data";  # Void context
        return;
    }
}

my @fruits = get_data();  # ('apple', 'banana', 'cherry')
my $count = get_data();   # 3
get_data();               # Prints: apple banana cherry
```

### Anonymous Subroutines and Closures

Perl supports first-class functions:

```perl
# Anonymous subroutine
my $greeter = sub {
    my ($name) = @_;
    say "Hello, $name!";
};
$greeter->('Perl');

# Closures capture variables
sub make_counter {
    my $count = 0;
    return sub { ++$count };
}

my $counter1 = make_counter();
my $counter2 = make_counter();
say $counter1->();  # 1
say $counter1->();  # 2
say $counter2->();  # 1 (independent counter)

# Higher-order functions
sub apply_to_list {
    my ($func, @list) = @_;
    return map { $func->($_) } @list;
}

my @doubled = apply_to_list(sub { $_ * 2 }, 1..5);
```

### Prototypes (Use with Caution)

Prototypes let you create subroutines that parse like built-ins:

```perl
# Prototype forces scalar context on first arg
sub my_push(\@@) {
    my ($array_ref, @values) = @_;
    push @$array_ref, @values;
}

my @stack;
my_push @stack, 1, 2, 3;  # No need for \@stack

# Block prototype for DSL-like syntax
sub with_file(&$) {
    my ($code, $filename) = @_;
    open my $fh, '<', $filename or die $!;
    $code->($fh);
    close $fh;
}

with_file {
    my $fh = shift;
    while (<$fh>) {
        print if /ERROR/;
    }
} 'logfile.txt';
```

## Error Handling: Die, Warn, and Eval

### Basic Error Handling

```perl
# die - throw an exception
open my $fh, '<', $file or die "Can't open $file: $!";

# warn - print warning but continue
warn "Config file not found, using defaults\n" unless -e $config;

# Custom die handler
$SIG{__DIE__} = sub {
    my $message = shift;
    log_error($message);
    die $message;  # Re-throw
};
```

### Exception Handling with eval

```perl
# Basic eval
eval {
    risky_operation();
    another_risky_operation();
};
if ($@) {
    warn "Operation failed: $@";
    # Handle error
}

# String eval (compile and run code at runtime)
my $code = 'print "Hello, World!\n"';
eval $code;
die "Code compilation failed: $@" if $@;
```

### Modern Exception Handling with Try::Tiny

```perl
use Try::Tiny;

try {
    risky_operation();
} catch {
    warn "Caught error: $_";
    # $_ contains the error
} finally {
    cleanup();  # Always runs
};
```

## Practical Example: Log File Processor

Let's combine everything into a real-world script:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';
use Try::Tiny;
use Time::Piece;

# Configuration
my %severity_levels = (
    DEBUG   => 0,
    INFO    => 1,
    WARNING => 2,
    ERROR   => 3,
    FATAL   => 4,
);

# Process command line
my ($logfile, $min_level) = @ARGV;
die "Usage: $0 <logfile> [min_level]\n" unless $logfile;
$min_level //= 'INFO';

# Main processing
process_log($logfile, $min_level);

sub process_log($file, $min_level) {
    my $min_severity = $severity_levels{$min_level}
        // die "Unknown level: $min_level";

    open my $fh, '<', $file or die "Can't open $file: $!";

    my %stats;
    my $line_count = 0;

    LINE: while (my $line = <$fh>) {
        $line_count++;
        chomp $line;

        # Skip empty lines and comments
        next LINE if $line =~ /^\s*$/;
        next LINE if $line =~ /^\s*#/;

        # Parse log line
        my ($timestamp, $level, $message) = parse_line($line);
        next LINE unless $timestamp;  # Skip unparseable lines

        # Filter by severity
        my $severity = $severity_levels{$level} // 0;
        next LINE if $severity < $min_severity;

        # Collect statistics
        $stats{$level}++;

        # Special handling for errors
        if ($level eq 'ERROR' || $level eq 'FATAL') {
            handle_error($timestamp, $level, $message);
        }
    }

    close $fh;

    # Report statistics
    report_stats(\%stats, $line_count);
}

sub parse_line($line) {
    # Example format: 2024-01-15 10:30:45 [ERROR] Connection timeout
    return unless $line =~ /
        ^(\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2})\s+  # timestamp
        \[(\w+)\]\s+                                   # level
        (.+)$                                          # message
    /x;

    return ($1, $2, $3);
}

sub handle_error($timestamp, $level, $message) {
    state %recent_errors;
    state $last_cleanup = time;

    # Clean old errors every minute
    if (time - $last_cleanup > 60) {
        %recent_errors = ();
        $last_cleanup = time;
    }

    # Detect repeated errors
    my $key = "$level:$message";
    $recent_errors{$key}++;

    if ($recent_errors{$key} == 1) {
        say "[$timestamp] $level: $message";
    } elsif ($recent_errors{$key} == 10) {
        warn "Error '$message' has occurred 10 times!";
    }
}

sub report_stats($stats, $total) {
    say "\n" . "=" x 40;
    say "Log Analysis Summary";
    say "=" x 40;
    say "Total lines processed: $total";
    say "\nEvents by severity:";

    for my $level (sort { $severity_levels{$b} <=> $severity_levels{$a} }
                   keys %severity_levels) {
        my $count = $stats->{$level} // 0;
        next unless $count;
        printf "  %-8s: %6d\n", $level, $count;
    }
}

__DATA__
2024-01-15 10:30:45 [INFO] Server started
2024-01-15 10:30:46 [DEBUG] Loading configuration
2024-01-15 10:31:00 [ERROR] Database connection failed
2024-01-15 10:31:01 [ERROR] Database connection failed
2024-01-15 10:31:02 [WARNING] Retrying database connection
```

## Best Practices

1. **Use statement modifiers for simple conditions** - They make code more readable
2. **Prefer foreach over C-style for** - Unless you specifically need the index
3. **Use map and grep for transformations** - They're faster and clearer than loops
4. **Always unpack @_ at the start of subroutines** - Makes the interface clear
5. **Use state variables instead of file-scoped variables** - Better encapsulation
6. **Handle errors early and explicitly** - Don't let them propagate silently

## Coming Up Next

Now that you understand Perl's control flow and subroutines, you're ready for the main event: regular expressions. In the next chapter, we'll explore why Perl's regex implementation is still the gold standard, and how to wield this power effectively in your system administration tasks.

---

*Remember: Good Perl code tells a story. The statement modifiers, flexible loops, and rich error handling aren't just features—they're tools for expressing your intent clearly. Use them to write code that reads like documentation.*