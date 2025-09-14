# Chapter 3: Perl Fundamentals - Variables and Data Types

> "In Perl, the variable tells you what it contains, not what it is." - Larry Wall

If you're coming from other languages, Perl's approach to variables might seem... different. Where Python has lists and dicts, and C has int and char, Perl has scalars, arrays, and hashes. But here's the thing: this simplicity is deceptive. These three data types, combined with references and context, give you incredible flexibility. Let's explore.

## The Sigils: Perl's Type System

In Perl, variables wear their type on their sleeve—literally. The symbol at the beginning of a variable (called a sigil) tells you what kind of data it holds:

- `$` - Scalar (single value)
- `@` - Array (ordered list)
- `%` - Hash (key-value pairs)

This isn't just syntax; it's documentation. When you see `$name`, you know it's a single value. When you see `@names`, you know it's a list. No guessing, no IDE required.

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

my $server = 'web01.prod';           # Scalar: single value
my @servers = qw(web01 web02 web03); # Array: list of values
my %status = (                       # Hash: key-value pairs
    web01 => 'running',
    web02 => 'running',
    web03 => 'maintenance'
);
```

## Scalars: Not Just Simple Values

The name "scalar" suggests simplicity, but Perl scalars are surprisingly sophisticated. A scalar can hold:

```perl
my $integer = 42;
my $float = 3.14159;
my $string = "Hello, World";
my $huge_number = 123_456_789;  # Underscores for readability
my $binary = 0b11111111;         # Binary literal (255)
my $hex = 0xFF;                  # Hexadecimal literal (255)
my $octal = 0755;                # Octal literal (493)

# Perl converts between types automatically
my $answer = "42";
my $result = $answer + 8;  # 50 - Perl converts string to number
say "The answer is $result";

# But be careful with strings that don't look like numbers
my $name = "Larry";
my $bad_math = $name + 5;  # 5 - "Larry" becomes 0 in numeric context
```

### The Magic of Interpolation

One of Perl's most loved features is string interpolation:

```perl
my $user = 'alice';
my $home = '/home';
my $path = "$home/$user";  # Double quotes interpolate
say "User $user home: $path";

# But single quotes don't interpolate
my $literal = '$home/$user';  # Literally: $home/$user
say 'This is literal: $literal';  # Prints: This is literal: $literal

# Need a literal $ in double quotes? Escape it
say "The cost is \$42.00";

# Complex expressions need curly braces
my $count = 5;
say "Next value: ${count}1";     # Next value: 51
say "Squared: @{[$count**2]}";   # Squared: 25 (array interpolation trick)
```

### Undefined Values and Truth

Perl has a special value `undef` that represents "no value":

```perl
my $nothing;  # $nothing is undef
my $something = undef;  # Explicitly undef

# Check for definedness
if (defined $nothing) {
    say "This won't print";
}

# Perl's truth rules:
# False: undef, 0, "0", "" (empty string), empty list
# True: Everything else (including "00", "0.0", negative numbers)

my @falsy_values = (undef, 0, "0", "");
for my $val (@falsy_values) {
    say "Value " . ($val // 'undef') . " is false" unless $val;
}

# The // operator (defined-or) is incredibly useful
my $port = $ENV{PORT} // 8080;  # Use env var or default to 8080
```

## Arrays: Lists with Attitude

Perl arrays are ordered, integer-indexed collections. But they're more flexible than arrays in most languages:

```perl
# Multiple ways to create arrays
my @empty = ();
my @numbers = (1, 2, 3, 4, 5);
my @words = qw(apple banana cherry);  # qw = quote words
my @mixed = (42, "hello", 3.14, undef);  # Mixed types? No problem!

# Array operations
push @numbers, 6;           # Add to end
my $last = pop @numbers;    # Remove and return last element
unshift @numbers, 0;        # Add to beginning
my $first = shift @numbers; # Remove and return first element

# Array access
say $numbers[0];   # First element (note the $)
say $numbers[-1];  # Last element (negative indexing!)
say $numbers[-2];  # Second to last

# Array slices
my @subset = @numbers[1..3];     # Elements 1, 2, 3
my @selection = @numbers[0, 2, 4]; # Elements 0, 2, 4

# Array size
my $size = @numbers;        # Array in scalar context gives size
my $last_index = $#numbers; # Last valid index

# Useful array operations
my @sorted = sort @words;
my @reversed = reverse @numbers;
my @unique = do { my %seen; grep { !$seen{$_}++ } @mixed };
```

### The Power of List Context

Perl's context sensitivity is unique. The same expression can return different values depending on context:

```perl
my @lines = <DATA>;  # List context: all lines
my $line = <DATA>;   # Scalar context: one line

# Many functions are context-aware
my @matches = $text =~ /\w+/g;  # List context: all matches
my $count = $text =~ /\w+/g;    # Scalar context: count of matches

# Force context
my $count = @array;          # Scalar context
my @copy = @{[@array]};      # List context (array ref then deref)
my ($first) = @array;        # List context, but only taking first element
```

## Hashes: The Swiss Army Data Structure

Hashes (associative arrays) map keys to values. They're unordered but incredibly fast for lookups:

```perl
# Creating hashes
my %empty = ();
my %config = (
    host => 'localhost',
    port => 8080,
    ssl  => 1,
);

# Fat comma => is just a fancy comma that quotes the left side
my %same_config = ('host', 'localhost', 'port', 8080, 'ssl', 1);

# Hash operations
$config{timeout} = 30;           # Add/update
my $host = $config{host};        # Access (note the $)
delete $config{ssl};             # Remove key
my $exists = exists $config{port}; # Check if key exists

# Hash slices
my @values = @config{qw(host port)};  # Get multiple values
@config{qw(user pass)} = qw(admin secret); # Set multiple values

# Iterating hashes
for my $key (keys %config) {
    say "$key: $config{$key}";
}

# Or with each (stateful iterator)
while (my ($key, $value) = each %config) {
    say "$key => $value";
}

# Getting all keys and values
my @keys = keys %config;
my @values = values %config;
```

### Real-World Hash Example: Log Analysis

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Count occurrences of each IP in a log
my %ip_count;
while (<>) {  # Read from files or STDIN
    if (/(\d+\.\d+\.\d+\.\d+)/) {
        $ip_count{$1}++;
    }
}

# Sort by count and display top 10
my @sorted = sort { $ip_count{$b} <=> $ip_count{$a} } keys %ip_count;
for my $ip (@sorted[0..9]) {
    last unless defined $ip;
    printf "%15s: %d hits\n", $ip, $ip_count{$ip};
}
```

## References: Perl's Pointers

References let you create complex data structures. Think of them as pointers that are actually safe to use:

```perl
# Creating references
my @array = (1, 2, 3);
my $array_ref = \@array;        # Reference to array
my $anon_array = [1, 2, 3];     # Anonymous array ref

my %hash = (a => 1, b => 2);
my $hash_ref = \%hash;          # Reference to hash
my $anon_hash = {a => 1, b => 2}; # Anonymous hash ref

# Dereferencing
my @copy = @$array_ref;         # Old style
my @copy2 = @{$array_ref};      # With braces for clarity
my @copy3 = $array_ref->@*;     # Postfix dereference (modern)

# Arrow operator for accessing elements
say $array_ref->[0];            # First element
say $hash_ref->{a};             # Value for key 'a'

# Complex data structures
my $servers = {
    production => {
        web => ['web01', 'web02', 'web03'],
        db  => ['db01', 'db02'],
    },
    staging => {
        web => ['staging-web01'],
        db  => ['staging-db01'],
    },
};

# Access nested data
say $servers->{production}{web}[0];  # web01
push @{$servers->{production}{web}}, 'web04';  # Add web04

# Check structure with Data::Dumper
use Data::Dumper;
print Dumper($servers);
```

## Type Checking and Conversion

Perl is dynamically typed but not weakly typed. It knows what type of data you have:

```perl
use Scalar::Util qw(looks_like_number blessed reftype);

my $maybe_number = "42";
if (looks_like_number($maybe_number)) {
    say "It's a number!";
}

my $ref = [1, 2, 3];
say "Reference type: " . ref($ref);  # ARRAY

# Checking reference types
if (ref($ref) eq 'ARRAY') {
    say "It's an array reference";
}

# More sophisticated type checking
use Ref::Util qw(is_arrayref is_hashref is_coderef);
if (is_arrayref($ref)) {
    say "Definitely an array ref";
}
```

## Context: The Secret Sauce

Context is Perl's superpower. Every operation happens in either scalar or list context:

```perl
# The same function can return different things
sub context_aware {
    my @data = (1, 2, 3, 4, 5);
    return wantarray ? @data : scalar(@data);
}

my @list = context_aware();  # (1, 2, 3, 4, 5)
my $scalar = context_aware(); # 5

# Forcing context
my $count = () = $string =~ /pattern/g;  # Force list context, get count

# Context in file operations
my @lines = <$fh>;  # Read all lines
my $line = <$fh>;   # Read one line

# The grep operator is context-aware
my @matches = grep { $_ > 5 } @numbers;  # List of matches
my $count = grep { $_ > 5 } @numbers;    # Count of matches
```

## Special Variables: Perl's Magic

Perl has numerous special variables that make common tasks easier:

```perl
# $_ - The default variable
for (1..5) {
    say;  # Prints $_ by default
}

# @_ - Subroutine arguments
sub greet {
    my ($name) = @_;  # Common idiom
    say "Hello, $name!";
}

# $! - System error message
open my $fh, '<', 'nonexistent.txt' or die "Can't open: $!";

# $? - Child process exit status
system('ls');
say "Command failed: $?" if $?;

# $$ - Process ID
say "My PID is $$";

# $0 - Program name
say "Running $0";

# @ARGV - Command line arguments
say "Arguments: @ARGV";

# %ENV - Environment variables
say "Home directory: $ENV{HOME}";
```

## Practical Example: Server Status Monitor

Let's put it all together with a practical script:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Time::Piece;

# Configuration
my %servers = (
    'web01.prod'    => { ip => '10.0.1.10', service => 'nginx' },
    'web02.prod'    => { ip => '10.0.1.11', service => 'nginx' },
    'db01.prod'     => { ip => '10.0.2.10', service => 'mysql' },
    'cache01.prod'  => { ip => '10.0.3.10', service => 'redis' },
);

# Check each server
my @down_servers;
my $check_time = localtime;

for my $hostname (sort keys %servers) {
    my $server = $servers{$hostname};
    my $result = `ping -c 1 -W 1 $server->{ip} 2>&1`;

    if ($? != 0) {
        push @down_servers, $hostname;
        $server->{status} = 'down';
        $server->{last_seen} = 'unknown';
    } else {
        $server->{status} = 'up';
        $server->{last_seen} = $check_time->strftime('%Y-%m-%d %H:%M:%S');
    }
}

# Report
say "=" x 50;
say "Server Status Report - $check_time";
say "=" x 50;

for my $hostname (sort keys %servers) {
    my $s = $servers{$hostname};
    my $status_emoji = $s->{status} eq 'up' ? '✓' : '✗';
    printf "%-15s %-15s %-10s %s\n",
        $hostname, $s->{ip}, $s->{service}, $status_emoji;
}

if (@down_servers) {
    say "\n⚠️  Alert: " . scalar(@down_servers) . " servers down!";
    say "  - $_" for @down_servers;
}
```

## Key Takeaways

1. **Sigils are your friends**: They make code self-documenting
2. **Context matters**: Understanding scalar vs list context is crucial
3. **References enable complexity**: But start simple
4. **Interpolation saves time**: But know when to use single quotes
5. **Special variables are powerful**: But document their use

In the next chapter, we'll explore control flow and subroutines, where Perl's philosophy of "there's more than one way to do it" really shines. You'll learn about Perl's unique statement modifiers, the various loop constructs, and how to write subroutines that are both powerful and maintainable.

---

*Remember: In Perl, the data structure you choose shapes how you think about the problem. Choose wisely, but don't overthink it. You can always refactor later.*