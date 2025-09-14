# Chapter 5: Regular Expressions - Perl's Superpower

> "Some people, when confronted with a problem, think 'I know, I'll use regular expressions.' Now they have two problems." - Jamie Zawinski

> "Those people weren't using Perl." - A Perl Programmer

Regular expressions aren't bolted onto Perl as an afterthought or imported from a library. They're woven into the language's DNA. When other languages were struggling with clunky regex APIs, Perl developers were parsing complex log files with one-liners. This chapter will show you why Perl's regex implementation is still unmatched and how to wield this power responsibly.

## The Basics (But Better)

### Match Operator: m//

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

my $text = "The server at 192.168.1.100 responded in 245ms";

# Basic matching
if ($text =~ /server/) {
    say "Found 'server'";
}

# Capture groups
if ($text =~ /(\d+\.\d+\.\d+\.\d+)/) {
    say "IP address: $1";  # $1 contains first capture
}

# Multiple captures
if ($text =~ /at ([\d.]+) responded in (\d+)ms/) {
    my ($ip, $time) = ($1, $2);
    say "Server $ip took ${time}ms";
}

# The !~ operator for negation
say "No errors!" if $text !~ /error|fail|timeout/i;

# Default variable $_
$_ = "Testing 123";
say "Contains number" if /\d+/;  # No need for $_ =~
```

### Substitution Operator: s///

```perl
my $config = "ServerName = localhost:8080";

# Basic substitution
$config =~ s/localhost/127.0.0.1/;
say $config;  # ServerName = 127.0.0.1:8080

# Global substitution with /g
my $log = "Error Error Warning Error";
$log =~ s/Error/Issue/g;
say $log;  # Issue Issue Warning Issue

# Capture and replace
my $date = "2024-01-15";
$date =~ s/(\d{4})-(\d{2})-(\d{2})/$3\/$2\/$1/;
say $date;  # 15/01/2024

# Using the result
my $count = $log =~ s/Warning/ALERT/g;  # Returns number of replacements
say "Replaced $count warnings";

# The /r modifier returns modified string without changing original
my $original = "hello world";
my $modified = $original =~ s/world/Perl/r;
say $original;  # hello world (unchanged)
say $modified;  # hello Perl
```

### The Transliteration Operator: tr/// (or y///)

Not technically a regex, but often used alongside them:

```perl
my $text = "Hello World 123";

# Count characters
my $digit_count = $text =~ tr/0-9//;
say "Contains $digit_count digits";

# ROT13 cipher
$text =~ tr/A-Za-z/N-ZA-Mn-za-m/;
say $text;  # Uryyb Jbeyq 123

# Remove duplicates
$text =~ tr/a-z//s;  # /s squashes duplicate characters

# Delete characters
$text =~ tr/0-9//d;  # /d deletes matched characters
```

## Regex Modifiers: Changing the Rules

```perl
# /i - Case insensitive
say "Match!" if "HELLO" =~ /hello/i;

# /x - Extended formatting (ignore whitespace, allow comments)
my $ip_regex = qr/
    ^                  # Start of string
    (\d{1,3})         # First octet
    \.                # Literal dot
    (\d{1,3})         # Second octet
    \.                # Literal dot
    (\d{1,3})         # Third octet
    \.                # Literal dot
    (\d{1,3})         # Fourth octet
    $                  # End of string
/x;

# /s - Single line mode (. matches newline)
my $html = "<div>\nContent\n</div>";
$html =~ /<div>(.*?)<\/div>/s;  # Captures across newlines

# /m - Multi-line mode (^ and $ match line boundaries)
my $multi = "Line 1\nLine 2\nLine 3";
my @lines = $multi =~ /^Line \d+$/gm;

# /g - Global matching
my $data = "cat bat rat";
my @words = $data =~ /\w+/g;  # ('cat', 'bat', 'rat')

# /o - Compile pattern once (optimization for loops)
for my $line (@huge_file) {
    $line =~ /$pattern/o;  # Pattern compiled only once
}
```

## Advanced Pattern Matching

### Non-Capturing Groups

```perl
# (?:...) doesn't create a capture variable
my $url = "https://www.example.com:8080/path";

if ($url =~ /^(https?):\/\/(?:www\.)?([^:\/]+)(?::(\d+))?/) {
    my ($protocol, $domain, $port) = ($1, $2, $3);
    $port //= $protocol eq 'https' ? 443 : 80;
    say "Protocol: $protocol, Domain: $domain, Port: $port";
}
```

### Named Captures (Perl 5.10+)

```perl
# Much more readable than $1, $2, $3...
my $log_line = '2024-01-15 10:30:45 [ERROR] Connection timeout';

if ($log_line =~ /
    (?<date>\d{4}-\d{2}-\d{2})\s+
    (?<time>\d{2}:\d{2}:\d{2})\s+
    \[(?<level>\w+)\]\s+
    (?<message>.+)
/x) {
    say "Date: $+{date}";
    say "Time: $+{time}";
    say "Level: $+{level}";
    say "Message: $+{message}";
}
```

### Lookahead and Lookbehind

```perl
# Positive lookahead (?=...)
# Match 'test' only if followed by 'ing'
"testing tested" =~ /test(?=ing)/;  # Matches 'test' in 'testing'

# Negative lookahead (?!...)
# Match 'test' only if NOT followed by 'ing'
"testing tested" =~ /test(?!ing)/;  # Matches 'test' in 'tested'

# Positive lookbehind (?<=...)
# Match numbers preceded by '$'
"Price: $50, €50" =~ /(?<=\$)\d+/;  # Matches '50' after '$'

# Negative lookbehind (?<!...)
# Match numbers NOT preceded by '$'
"Price: $50, €50" =~ /(?<!\$)\d+/;  # Matches '50' after '€'

# Practical example: Extract price without currency symbol
my $price_text = "The cost is $1,234.56 including tax";
if ($price_text =~ /\$(?<price>[\d,]+\.?\d*)/) {
    my $price = $+{price};
    $price =~ s/,//g;  # Remove commas
    say "Price: $price";  # Price: 1234.56
}
```

### Recursive Patterns

Perl can match nested structures:

```perl
# Match balanced parentheses
my $balanced = qr/
    \(                   # Opening paren
    (?:
        [^()]+          # Non-parens
        |
        (?R)            # Recurse entire pattern
    )*
    \)                   # Closing paren
/x;

my $text = "func(a, b(c, d(e)), f)";
say "Balanced!" if $text =~ /^func$balanced$/;
```

## Real-World Regex Patterns

### Email Validation (Simplified)

```perl
# This is simplified. Real email validation is complex!
my $email_regex = qr/
    ^                       # Start
    [\w\.\-]+              # Local part
    \@                      # At sign
    [\w\-]+                # Domain name
    (?:\.[\w\-]+)+         # Domain extensions
    $                       # End
/x;

my @emails = qw(
    user@example.com
    john.doe@company.co.uk
    invalid@
    @invalid.com
    valid+tag@gmail.com
);

for my $email (@emails) {
    if ($email =~ $email_regex) {
        say "$email is valid";
    } else {
        say "$email is invalid";
    }
}
```

### Log File Parsing

```perl
# Apache/Nginx log parser
my $log_regex = qr/
    ^
    (?<ip>[\d\.]+)\s+           # IP address
    (?<ident>\S+)\s+            # Identity
    (?<user>\S+)\s+             # User
    \[(?<timestamp>[^\]]+)\]\s+ # Timestamp
    "(?<request>[^"]+)"\s+      # Request
    (?<status>\d{3})\s+         # Status code
    (?<size>\d+|-)\s*           # Response size
    "(?<referer>[^"]*)"\s*      # Referer
    "(?<agent>[^"]*)"           # User agent
/x;

while (my $line = <$log_fh>) {
    next unless $line =~ $log_regex;

    my %entry = %+;  # Copy all named captures

    # Process the log entry
    if ($entry{status} >= 500) {
        warn "Server error: $entry{request} returned $entry{status}";
    }

    # Extract more info from request
    if ($entry{request} =~ /^(?<method>\S+)\s+(?<path>\S+)\s+(?<proto>\S+)/) {
        $entry{method} = $+{method};
        $entry{path} = $+{path};
    }
}
```

### Configuration File Parser

```perl
# Parse INI-style config files
sub parse_config {
    my ($filename) = @_;
    my %config;
    my $section = 'DEFAULT';

    open my $fh, '<', $filename or die "Can't open $filename: $!";

    while (my $line = <$fh>) {
        chomp $line;

        # Skip comments and empty lines
        next if $line =~ /^\s*(?:#|$)/;

        # Section header
        if ($line =~ /^\[([^\]]+)\]/) {
            $section = $1;
            next;
        }

        # Key-value pair
        if ($line =~ /
            ^\s*
            ([^=]+?)       # Key (non-greedy)
            \s*=\s*        # Equals with optional whitespace
            (.*)           # Value
            $
        /x) {
            my ($key, $value) = ($1, $2);

            # Remove quotes if present
            $value =~ s/^["']|["']$//g;

            # Store in config
            $config{$section}{$key} = $value;
        }
    }

    close $fh;
    return \%config;
}
```

## Performance and Optimization

### Compile Once, Use Many

```perl
# Bad: Regex compiled every iteration
for my $line (@lines) {
    if ($line =~ /$user_pattern/) {  # Compiles each time
        process($line);
    }
}

# Good: Pre-compile regex
my $regex = qr/$user_pattern/;
for my $line (@lines) {
    if ($line =~ $regex) {  # Already compiled
        process($line);
    }
}

# Better: Use state for persistent compiled regex
sub match_pattern {
    my ($text, $pattern) = @_;
    state %compiled;

    $compiled{$pattern} //= qr/$pattern/;
    return $text =~ $compiled{$pattern};
}
```

### Avoiding Backtracking

```perl
# Bad: Catastrophic backtracking possible
$text =~ /(\w+)*$/;  # Nested quantifiers

# Good: Possessive quantifiers prevent backtracking
$text =~ /(\w++)*$/;  # ++ is possessive

# Bad: Greedy matching with backtracking
$html =~ /<div>.*<\/div>/;  # Matches too much

# Good: Non-greedy matching
$html =~ /<div>.*?<\/div>/;  # *? is non-greedy

# Better: Explicit matching
$html =~ /<div>[^<]*<\/div>/;  # More efficient
```

## Practical Script: Web Scraper

Let's build a simple web scraper using Perl's regex powers:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use LWP::Simple;
use HTML::Entities;

# Fetch a web page and extract information
my $url = shift @ARGV or die "Usage: $0 <URL>\n";
my $html = get($url) or die "Couldn't fetch $url\n";

# Remove script and style blocks
$html =~ s/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>//gis;
$html =~ s/<style\b[^<]*(?:(?!<\/style>)<[^<]*)*<\/style>//gis;

# Extract title
my ($title) = $html =~ /<title>([^<]+)<\/title>/i;
$title //= 'No title';
say "Title: " . decode_entities($title);

# Extract all links
my @links = $html =~ /<a\s+(?:[^>]*?\s+)?href=["']([^"']+)["']/gi;
say "\nFound " . scalar(@links) . " links:";

# Process and display unique links
my %seen;
for my $link (@links) {
    next if $seen{$link}++;
    next if $link =~ /^#/;  # Skip anchors

    # Make relative URLs absolute
    if ($link !~ /^https?:\/\//) {
        if ($link =~ /^\//) {
            # Absolute path
            my ($base) = $url =~ /(https?:\/\/[^\/]+)/;
            $link = "$base$link";
        } else {
            # Relative path
            my ($base) = $url =~ /(.*\/)/;
            $link = "$base$link";
        }
    }

    say "  - $link";
}

# Extract meta tags
say "\nMeta tags:";
while ($html =~ /<meta\s+([^>]+)>/gi) {
    my $meta = $1;
    my ($name) = $meta =~ /name=["']([^"']+)["']/i;
    my ($content) = $meta =~ /content=["']([^"']+)["']/i;

    if ($name && $content) {
        say "  $name: " . decode_entities($content);
    }
}

# Extract all email addresses (naive pattern)
my @emails = $html =~ /\b([\w\.\-]+@[\w\.\-]+\.\w+)\b/g;
if (@emails) {
    say "\nEmail addresses found:";
    my %unique_emails;
    @unique_emails{@emails} = ();
    say "  - $_" for sort keys %unique_emails;
}
```

## Debugging Regular Expressions

### The use re 'debug' Pragma

```perl
use re 'debug';

"test string" =~ /test.*string/;

# This will output the regex compilation and execution process
# Great for understanding why a regex isn't matching
```

### Building Regexes Incrementally

```perl
# Start simple and build up
my $regex = qr/\d+/;           # Match numbers
$regex = qr/\d+\.\d+/;         # Match decimals
$regex = qr/\d+(?:\.\d+)?/;    # Optional decimal part
$regex = qr/^\d+(?:\.\d+)?$/;  # Anchor to whole string

# Test at each stage
my @test_cases = qw(123 123.45 .45 123. abc);
for my $test (@test_cases) {
    if ($test =~ $regex) {
        say "$test matches";
    } else {
        say "$test doesn't match";
    }
}
```

## Common Gotchas and Solutions

### The Greediness Problem

```perl
my $xml = '<tag>content</tag><tag>more</tag>';

# Wrong: Greedy matching
$xml =~ /<tag>.*<\/tag>/;  # Matches entire string!

# Right: Non-greedy
$xml =~ /<tag>.*?<\/tag>/;  # Matches first tag pair

# Better: Explicit
$xml =~ /<tag>[^<]*<\/tag>/;  # Most efficient
```

### The Anchor Trap

```perl
# Dangerous: No anchors
if ($input =~ /\d{3}/) {
    # Matches "abc123def" - probably not intended!
}

# Safe: With anchors
if ($input =~ /^\d{3}$/) {
    # Only matches exactly 3 digits
}
```

### Special Characters in Variables

```perl
my $user_input = "What???";

# Wrong: ? is a regex metacharacter
if ($text =~ /$user_input/) {  # Error!

# Right: Quote metacharacters
if ($text =~ /\Q$user_input\E/) {  # Treats ??? as literal
```

## Best Practices

1. **Comment complex regexes** - Use /x modifier liberally
2. **Name your captures** - $+{name} is clearer than $3
3. **Compile once when possible** - Use qr// for repeated patterns
4. **Test incrementally** - Build complex patterns step by step
5. **Consider alternatives** - Sometimes a parser is better than a regex
6. **Anchor when appropriate** - Prevent unexpected matches
7. **Be careful with user input** - Always use \Q...\E for literal matching

## The Zen of Perl Regexes

Regular expressions in Perl aren't just a feature—they're a philosophy. They embody Perl's core principle: make easy things easy and hard things possible. Yes, you can write unreadable regex golf. But you can also write clear, maintainable patterns that solve real problems elegantly.

The key is knowing when to use them. Not every text processing task needs a regex. But when you do need one, Perl ensures you have the full power of regular expressions at your fingertips, integrated seamlessly into the language.

---

*Next up: File I/O and directory operations. We'll see how Perl's "Do What I Mean" philosophy extends to file handling, and why Perl remains a favorite for system administrators who need to process thousands of files efficiently.*