# Chapter 7: Advanced Text Processing

> "Perl is the text surgeon's scalpel, awk is a butter knife, and sed is a club." - Randal Schwartz

You've mastered regular expressions. You can read and write files. Now let's combine these skills to do what Perl does best: transform text in ways that would make other languages weep. This chapter covers advanced parsing techniques, text generation, format conversion, and the dark art of writing your own mini-languages.

## Beyond Simple Matching: Parse::RecDescent

Sometimes regex isn't enough. When you need to parse complex, nested structures, you need a real parser:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Parse::RecDescent;

# Define a grammar for simple arithmetic
my $grammar = q{
    expression: term(s /[+-]/) {
        my $result = shift @{$item[1]};
        while (@{$item[1]}) {
            my $op = shift @{$item[1]};
            my $val = shift @{$item[1]};
            $result = $op eq '+' ? $result + $val : $result - $val;
        }
        $result;
    }

    term: factor(s /[*\/]/) {
        my $result = shift @{$item[1]};
        while (@{$item[1]}) {
            my $op = shift @{$item[1]};
            my $val = shift @{$item[1]};
            $result = $op eq '*' ? $result * $val : $result / $val;
        }
        $result;
    }

    factor: number | '(' expression ')' { $item[2] }

    number: /\d+(\.\d+)?/ { $item[1] }
};

my $parser = Parse::RecDescent->new($grammar);

# Test it
my @tests = (
    "2 + 3",
    "2 + 3 * 4",
    "(2 + 3) * 4",
    "10 / 2 - 3",
);

for my $expr (@tests) {
    my $result = $parser->expression($expr);
    say "$expr = $result";
}
```

## Text Tables: Making Data Pretty

### Using Text::Table

```perl
use Text::Table;

# Create a formatted table
my $table = Text::Table->new(
    "Server\n&left",
    "Status\n&center",
    "CPU %\n&right",
    "Memory\n&right",
    "Uptime\n&right"
);

# Add data
my @data = (
    ['web01', 'Running', '45%', '2.3GB', '45 days'],
    ['web02', 'Running', '67%', '3.1GB', '45 days'],
    ['db01',  'Warning', '89%', '7.8GB', '12 days'],
    ['cache01', 'Down',   'N/A', 'N/A',   'N/A'],
);

$table->load(@data);

# Print with rules
print $table->rule('-', '+');
print $table->title;
print $table->rule('-', '+');
print $table->body;
print $table->rule('-', '+');
```

### Creating ASCII Art Tables

```perl
sub create_ascii_table {
    my ($headers, $rows) = @_;

    # Calculate column widths
    my @widths;
    for my $i (0..$#$headers) {
        $widths[$i] = length($headers->[$i]);
        for my $row (@$rows) {
            my $len = length($row->[$i] // '');
            $widths[$i] = $len if $len > $widths[$i];
        }
    }

    # Build table
    my $separator = '+' . join('+', map { '-' x ($_ + 2) } @widths) . '+';
    my $format = '| ' . join(' | ', map { "%-${_}s" } @widths) . ' |';

    # Print table
    say $separator;
    printf "$format\n", @$headers;
    say $separator;
    for my $row (@$rows) {
        printf "$format\n", map { $_ // '' } @$row;
    }
    say $separator;
}

# Usage
create_ascii_table(
    ['Name', 'Age', 'City'],
    [
        ['Alice', 30, 'New York'],
        ['Bob',   25, 'Los Angeles'],
        ['Carol', 35, 'Chicago'],
    ]
);
```

## Template Processing

### Quick and Dirty Templates

```perl
# Simple variable substitution
sub process_template {
    my ($template, $vars) = @_;

    $template =~ s/\{\{(\w+)\}\}/$vars->{$1} // ''/ge;
    return $template;
}

my $template = <<'END';
Dear {{name}},

Your server {{server}} is currently {{status}}.
CPU usage: {{cpu}}%
Memory usage: {{memory}}%

Please take appropriate action.

Regards,
Monitoring System
END

my $output = process_template($template, {
    name   => 'Admin',
    server => 'web01',
    status => 'critical',
    cpu    => 95,
    memory => 87,
});

print $output;
```

### Template Toolkit (Professional Templates)

```perl
use Template;

my $tt = Template->new({
    INCLUDE_PATH => './templates',
    INTERPOLATE  => 1,
});

my $template = <<'END';
[% FOREACH server IN servers %]
Server: [% server.name %]
  Status: [% server.status %]
  Services:
  [% FOREACH service IN server.services %]
    - [% service %]: [% server.service_status.$service %]
  [% END %]
[% END %]

Summary:
  Total servers: [% servers.size %]
  Running: [% servers.grep('^status', 'running').size %]
  Issues: [% servers.grep('^status', 'warning|critical').size %]
END

my $vars = {
    servers => [
        {
            name => 'web01',
            status => 'running',
            services => ['nginx', 'php-fpm'],
            service_status => {
                'nginx' => 'active',
                'php-fpm' => 'active',
            },
        },
        {
            name => 'db01',
            status => 'warning',
            services => ['mysql'],
            service_status => {
                'mysql' => 'degraded',
            },
        },
    ],
};

$tt->process(\$template, $vars) or die $tt->error;
```

## Parsing Structured Text Formats

### Parsing Configuration Files

```perl
# Parse Apache-style config
sub parse_apache_config {
    my ($filename) = @_;
    my %config;
    my @context_stack;

    open my $fh, '<', $filename or die $!;
    while (<$fh>) {
        chomp;
        s/^\s+|\s+$//g;  # Trim
        next if /^#/ || /^$/;  # Skip comments and blanks

        # Directive with value
        if (/^(\w+)\s+(.+)$/) {
            my ($directive, $value) = ($1, $2);
            $value =~ s/^["']|["']$//g;  # Remove quotes

            if (@context_stack) {
                # Inside a context
                my $ref = \%config;
                $ref = $ref->{$_} for @context_stack;
                push @{$ref->{$directive}}, $value;
            } else {
                push @{$config{$directive}}, $value;
            }
        }
        # Context start
        elsif (/^<(\w+)(?:\s+(.+))?>$/) {
            my ($context, $param) = ($1, $2);
            push @context_stack, "$context:$param";

            my $ref = \%config;
            $ref = $ref->{$_} for @context_stack;
            $ref = {};
        }
        # Context end
        elsif (/^<\/(\w+)>$/) {
            pop @context_stack;
        }
    }
    close $fh;

    return \%config;
}
```

### Parsing Fixed-Width Records

```perl
# Parse mainframe-style fixed-width data
sub parse_fixed_width {
    my ($filename, $layout) = @_;
    my @records;

    open my $fh, '<', $filename or die $!;
    while (my $line = <$fh>) {
        chomp $line;
        my %record;

        for my $field (@$layout) {
            my ($name, $start, $length) = @$field;
            $record{$name} = substr($line, $start - 1, $length);
            $record{$name} =~ s/^\s+|\s+$//g;  # Trim
        }

        push @records, \%record;
    }
    close $fh;

    return \@records;
}

# Define layout
my $layout = [
    ['id',       1,  5],
    ['name',     6, 20],
    ['dept',    26, 15],
    ['salary',  41, 10],
    ['hired',   51, 10],
];

my $records = parse_fixed_width('employees.dat', $layout);
```

## Text Differences and Patching

### Finding Differences

```perl
use Text::Diff;

my $diff = diff('file1.txt', 'file2.txt', { STYLE => 'Unified' });
print $diff;

# Or compare strings
my $old = "Line 1\nLine 2\nLine 3\n";
my $new = "Line 1\nLine 2 modified\nLine 3\nLine 4\n";

my $diff = diff(\$old, \$new, { STYLE => 'Context' });
print $diff;

# Custom diff with Algorithm::Diff
use Algorithm::Diff qw(sdiff);

my @old = split /\n/, $old;
my @new = split /\n/, $new;

my @diff = sdiff(\@old, \@new);
for my $change (@diff) {
    my ($flag, $old_line, $new_line) = @$change;
    if ($flag eq 'u') {
        say "  $old_line";  # Unchanged
    } elsif ($flag eq 'c') {
        say "- $old_line";   # Changed from
        say "+ $new_line";   # Changed to
    } elsif ($flag eq '-') {
        say "- $old_line";   # Deleted
    } elsif ($flag eq '+') {
        say "+ $new_line";   # Added
    }
}
```

## Creating Domain-Specific Languages (DSLs)

### A Simple Query Language

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Define a simple query DSL
package QueryDSL;

sub new {
    my ($class) = @_;
    return bless { conditions => [] }, $class;
}

sub where {
    my ($self, $field) = @_;
    $self->{current_field} = $field;
    return $self;
}

sub equals {
    my ($self, $value) = @_;
    push @{$self->{conditions}}, {
        field => $self->{current_field},
        op => '=',
        value => $value,
    };
    return $self;
}

sub greater_than {
    my ($self, $value) = @_;
    push @{$self->{conditions}}, {
        field => $self->{current_field},
        op => '>',
        value => $value,
    };
    return $self;
}

sub and {
    my ($self) = @_;
    $self->{last_conjunction} = 'AND';
    return $self;
}

sub to_sql {
    my ($self) = @_;
    my @parts;
    for my $cond (@{$self->{conditions}}) {
        push @parts, "$cond->{field} $cond->{op} '$cond->{value}'";
    }
    return 'WHERE ' . join(' AND ', @parts);
}

package main;

# Use the DSL
my $query = QueryDSL->new()
    ->where('status')->equals('active')
    ->and
    ->where('age')->greater_than(18);

say $query->to_sql();  # WHERE status = 'active' AND age > '18'
```

### A Configuration DSL

```perl
# Create a readable configuration DSL
package ConfigDSL;
use Modern::Perl '2023';

our %CONFIG;

sub server($&) {
    my ($name, $block) = @_;
    local $CONFIG{_current_server} = $name;
    $CONFIG{servers}{$name} = {};
    $block->();
}

sub host($) {
    my ($hostname) = @_;
    my $server = $CONFIG{_current_server};
    $CONFIG{servers}{$server}{host} = $hostname;
}

sub port($) {
    my ($port) = @_;
    my $server = $CONFIG{_current_server};
    $CONFIG{servers}{$server}{port} = $port;
}

sub service($) {
    my ($service) = @_;
    my $server = $CONFIG{_current_server};
    push @{$CONFIG{servers}{$server}{services}}, $service;
}

sub import {
    my $caller = caller;
    no strict 'refs';
    *{"${caller}::server"} = \&server;
    *{"${caller}::host"} = \&host;
    *{"${caller}::port"} = \&port;
    *{"${caller}::service"} = \&service;
}

package main;
use ConfigDSL;

# Now we can write config like this:
server 'web01' => sub {
    host 'web01.example.com';
    port 8080;
    service 'nginx';
    service 'php-fpm';
};

server 'db01' => sub {
    host 'db01.example.com';
    port 3306;
    service 'mysql';
};

# Access the config
use Data::Dumper;
print Dumper(\%ConfigDSL::CONFIG);
```

## Text Analysis and Statistics

### Word Frequency Analysis

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

sub analyze_text {
    my ($text) = @_;

    # Basic statistics
    my $char_count = length($text);
    my $line_count = ($text =~ tr/\n//) + 1;
    my @sentences = split /[.!?]+/, $text;
    my $sentence_count = @sentences;

    # Word frequency
    my %word_freq;
    my $word_count = 0;
    while ($text =~ /\b(\w+)\b/g) {
        my $word = lc($1);
        $word_freq{$word}++;
        $word_count++;
    }

    # Calculate readability (Flesch Reading Ease approximation)
    my $avg_sentence_length = $word_count / ($sentence_count || 1);
    my $syllable_count = estimate_syllables($text);
    my $avg_syllables = $syllable_count / ($word_count || 1);

    my $flesch = 206.835
                - 1.015 * $avg_sentence_length
                - 84.6 * $avg_syllables;

    return {
        characters => $char_count,
        lines => $line_count,
        sentences => $sentence_count,
        words => $word_count,
        unique_words => scalar(keys %word_freq),
        avg_word_length => $char_count / ($word_count || 1),
        readability => $flesch,
        top_words => get_top_words(\%word_freq, 10),
    };
}

sub estimate_syllables {
    my ($text) = @_;
    my $count = 0;
    while ($text =~ /\b(\w+)\b/g) {
        my $word = lc($1);
        # Simple estimation: count vowel groups
        my $syllables = () = $word =~ /[aeiou]+/g;
        $syllables = 1 if $syllables == 0;
        $count += $syllables;
    }
    return $count;
}

sub get_top_words {
    my ($freq, $n) = @_;
    my @sorted = sort { $freq->{$b} <=> $freq->{$a} } keys %$freq;
    return [ map { { word => $_, count => $freq->{$_} } }
             @sorted[0..min($n-1, $#sorted)] ];
}

sub min { $_[0] < $_[1] ? $_[0] : $_[1] }

# Test it
my $sample = <<'END';
Perl is a high-level, general-purpose programming language.
It was originally developed by Larry Wall in 1987. Perl is
known for its text processing capabilities and is often
called the "Swiss Army chainsaw" of scripting languages.
END

my $stats = analyze_text($sample);
use Data::Dumper;
print Dumper($stats);
```

## Advanced String Manipulation

### Levenshtein Distance (Edit Distance)

```perl
use Text::Levenshtein qw(distance);

# Find similar strings
sub find_similar {
    my ($target, $candidates, $threshold) = @_;
    $threshold //= 3;

    my @similar;
    for my $candidate (@$candidates) {
        my $dist = distance($target, $candidate);
        push @similar, { string => $candidate, distance => $dist }
            if $dist <= $threshold;
    }

    return [ sort { $a->{distance} <=> $b->{distance} } @similar ];
}

my @commands = qw(start stop restart status enable disable);
my $user_input = 'statsu';  # Typo

my $similar = find_similar($user_input, \@commands);
if (@$similar) {
    say "Did you mean: " . $similar->[0]{string} . "?";
}
```

### Fuzzy String Matching

```perl
use String::Approx qw(amatch);

# Find approximate matches
my @files = glob("*.txt");
my $pattern = 'confg';  # Looking for 'config'

my @matches = amatch($pattern, ['i', '10%'], @files);
say "Possible matches for '$pattern':";
say "  $_" for @matches;

# Custom fuzzy search
sub fuzzy_grep {
    my ($pattern, $list, $tolerance) = @_;
    $tolerance //= 0.2;  # 20% difference allowed

    my $max_dist = int(length($pattern) * $tolerance);
    return find_similar($pattern, $list, $max_dist);
}
```

## Practical Example: Log Analysis Pipeline

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

# Pluggable log analysis pipeline
package LogPipeline;

sub new($class) {
    return bless {
        filters => [],
        extractors => [],
        aggregators => [],
    }, $class;
}

sub add_filter($self, $filter) {
    push @{$self->{filters}}, $filter;
    return $self;
}

sub add_extractor($self, $extractor) {
    push @{$self->{extractors}}, $extractor;
    return $self;
}

sub add_aggregator($self, $aggregator) {
    push @{$self->{aggregators}}, $aggregator;
    return $self;
}

sub process($self, $filename) {
    my @records;

    open my $fh, '<', $filename or die $!;
    LINE: while (my $line = <$fh>) {
        chomp $line;

        # Apply filters
        for my $filter (@{$self->{filters}}) {
            next LINE unless $filter->($line);
        }

        # Extract data
        my %record;
        for my $extractor (@{$self->{extractors}}) {
            my $data = $extractor->($line);
            %record = (%record, %$data) if $data;
        }

        push @records, \%record if %record;
    }
    close $fh;

    # Aggregate results
    my %results;
    for my $aggregator (@{$self->{aggregators}}) {
        my ($name, $value) = $aggregator->(\@records);
        $results{$name} = $value;
    }

    return \%results;
}

package main;

# Create pipeline
my $pipeline = LogPipeline->new();

# Add filters
$pipeline->add_filter(sub($line) {
    return $line !~ /^#/;  # Skip comments
});

$pipeline->add_filter(sub($line) {
    return $line =~ /ERROR|WARNING/;  # Only errors and warnings
});

# Add extractors
$pipeline->add_extractor(sub($line) {
    if ($line =~ /^(\S+)\s+(\S+)\s+\[([^\]]+)\]\s+(.+)/) {
        return {
            date => $1,
            time => $2,
            level => $3,
            message => $4,
        };
    }
    return undef;
});

$pipeline->add_extractor(sub($line) {
    if ($line =~ /user[_\s]+(\w+)/i) {
        return { user => $1 };
    }
    return undef;
});

# Add aggregators
$pipeline->add_aggregator(sub($records) {
    return ('total_errors', scalar(@$records));
});

$pipeline->add_aggregator(sub($records) {
    my %by_level;
    $by_level{$_->{level}}++ for @$records;
    return ('by_level', \%by_level);
});

$pipeline->add_aggregator(sub($records) {
    my %by_user;
    $by_user{$_->{user}}++ for grep { $_->{user} } @$records;
    return ('by_user', \%by_user);
});

# Process log file
my $results = $pipeline->process('application.log');

# Display results
say "Log Analysis Results:";
say "Total errors/warnings: $results->{total_errors}";
say "\nBy level:";
for my $level (sort keys %{$results->{by_level}}) {
    say "  $level: $results->{by_level}{$level}";
}
say "\nBy user:";
for my $user (sort keys %{$results->{by_user}}) {
    say "  $user: $results->{by_user}{$user}";
}
```

## Performance Tips for Text Processing

1. **Compile regexes once** - Use qr// for repeated patterns
2. **Avoid unnecessary captures** - Use (?:...) for grouping
3. **Process line by line** - Don't slurp huge files unless necessary
4. **Use index() for simple searches** - It's faster than regex for literals
5. **Consider Text::CSV_XS** - Much faster than pure Perl CSV parsing
6. **Profile your code** - Use Devel::NYTProf to find bottlenecks
7. **Use state variables** - For data that persists between function calls
8. **Benchmark alternatives** - Sometimes split is faster than regex

## Best Practices

1. **Make parsers modular** - Separate lexing, parsing, and semantic analysis
2. **Handle edge cases** - Empty input, malformed data, encoding issues
3. **Provide useful error messages** - Include line numbers and context
4. **Document your grammars** - Especially for complex parsers
5. **Test with real data** - Synthetic test data often misses edge cases
6. **Consider existing modules** - CPAN likely has what you need
7. **Use the right tool** - Not everything needs a full parser

## Conclusion

Advanced text processing is where Perl truly shines. Whether you're parsing complex formats, generating reports, or building your own languages, Perl provides the tools to do it elegantly and efficiently. The key is knowing which tool to use for each job.

Remember: text processing isn't just about regex. It's about understanding structure, extracting meaning, and transforming data. Perl gives you the flexibility to approach each problem in the way that makes most sense.

---

*Next: Working with structured data formats. We'll explore how Perl handles CSV, JSON, and XML, turning messy data into actionable information.*