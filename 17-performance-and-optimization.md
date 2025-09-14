# Chapter 17: Performance and Optimization

> "Premature optimization is the root of all evil, but that doesn't mean we should write slow code on purpose." - Modern Perl Wisdom

Performance matters when you're processing gigabytes of logs, handling thousands of requests, or running scripts hundreds of times per day. This chapter shows you how to identify bottlenecks, optimize critical paths, and make your Perl code fly. We'll cover profiling, benchmarking, and proven optimization techniques that actually make a difference.

## Profiling: Finding the Bottlenecks

### Using Devel::NYTProf

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Run with profiling:
# perl -d:NYTProf script.pl
# nytprofhtml
# open nytprof/index.html

# Example slow code to profile
sub process_data {
    my ($data) = @_;
    my @results;

    for my $item (@$data) {
        # Inefficient: regex compilation in loop
        if ($item =~ /$ENV{PATTERN}/) {
            push @results, transform($item);
        }
    }

    return \@results;
}

sub transform {
    my ($item) = @_;

    # Inefficient: repeated string concatenation
    my $result = "";
    for my $char (split //, $item) {
        $result .= process_char($char);
    }

    return $result;
}

sub process_char {
    my ($char) = @_;
    # Simulate expensive operation
    return uc($char);
}

# Profile programmatically
use Devel::NYTProf::Run;

Devel::NYTProf::Run::enable_profile();

# Code to profile
my @data = map { "test_string_$_" } 1..10000;
my $results = process_data(\@data);

Devel::NYTProf::Run::disable_profile();
Devel::NYTProf::Run::finish_profile();
```

### Benchmarking with Benchmark

```perl
use Benchmark qw(cmpthese timethese);
use Benchmark ':hireswallclock';  # High-resolution timing

# Compare different approaches
cmpthese(-3, {  # Run for 3 seconds
    'method1' => sub {
        my $str = "";
        $str .= "x" for 1..1000;
    },
    'method2' => sub {
        my $str = "x" x 1000;
    },
    'method3' => sub {
        my @parts = ("x") x 1000;
        my $str = join '', @parts;
    },
});

# Output:
#            Rate method1 method3 method2
# method1   523/s      --    -48%    -97%
# method3  1010/s     93%      --    -94%
# method2 17544/s   3255%   1637%      --

# Detailed timing
my $results = timethese(10000, {
    'regex' => sub {
        my $text = "Hello World";
        $text =~ s/World/Perl/;
    },
    'substr' => sub {
        my $text = "Hello World";
        substr($text, 6, 5, 'Perl');
    },
});

# Compare specific operations
use Benchmark qw(timeit timestr);

my $t = timeit(1000000, sub {
    my @array = 1..100;
    my $sum = 0;
    $sum += $_ for @array;
});

say "Time: " . timestr($t);
```

## Common Performance Optimizations

### String Operations

```perl
# SLOW: String concatenation in loop
sub slow_concat {
    my $result = "";
    for my $item (@_) {
        $result .= process($item);
        $result .= "\n";
    }
    return $result;
}

# FAST: Join array
sub fast_concat {
    my @results;
    for my $item (@_) {
        push @results, process($item);
    }
    return join("\n", @results);
}

# FASTER: Map
sub faster_concat {
    return join("\n", map { process($_) } @_);
}

# SLOW: Character-by-character processing
sub slow_reverse {
    my ($str) = @_;
    my $result = "";
    for my $char (split //, $str) {
        $result = $char . $result;
    }
    return $result;
}

# FAST: Built-in reverse
sub fast_reverse {
    my ($str) = @_;
    return scalar reverse $str;
}
```

### Regular Expression Optimization

```perl
# SLOW: Regex compilation in loop
sub slow_matching {
    my ($data, $pattern) = @_;
    my @matches;

    for my $item (@$data) {
        push @matches, $item if $item =~ /$pattern/;  # Compiles each time
    }

    return \@matches;
}

# FAST: Pre-compiled regex
sub fast_matching {
    my ($data, $pattern) = @_;
    my $regex = qr/$pattern/;  # Compile once
    my @matches;

    for my $item (@$data) {
        push @matches, $item if $item =~ $regex;
    }

    return \@matches;
}

# FASTER: Grep
sub faster_matching {
    my ($data, $pattern) = @_;
    my $regex = qr/$pattern/;
    return [ grep /$regex/, @$data ];
}

# Optimize complex regexes
my $email_regex = qr/
    \A                    # Start of string
    [\w\.\-]+            # Local part
    \@                   # At sign
    [\w\-]+              # Domain name
    (?:\.[\w\-]+)+       # Domain extensions
    \z                   # End of string
/x;  # /x for readability

# Use atomic groups to prevent backtracking
my $no_backtrack = qr/
    \A
    (?> \d+ )           # Atomic group - no backtracking
    \.
    (?> \d+ )
    \z
/x;
```

### Data Structure Optimization

```perl
# SLOW: Repeated hash lookups
sub slow_processing {
    my ($data) = @_;
    my %lookup;

    for my $item (@$data) {
        if (exists $lookup{$item->{id}}) {
            $lookup{$item->{id}}{count}++;
            $lookup{$item->{id}}{total} += $item->{value};
        } else {
            $lookup{$item->{id}} = {
                count => 1,
                total => $item->{value},
            };
        }
    }

    return \%lookup;
}

# FAST: Single lookup with reference
sub fast_processing {
    my ($data) = @_;
    my %lookup;

    for my $item (@$data) {
        my $entry = $lookup{$item->{id}} //= { count => 0, total => 0 };
        $entry->{count}++;
        $entry->{total} += $item->{value};
    }

    return \%lookup;
}

# Use arrays instead of hashes when possible
# SLOW: Hash for fixed set of keys
sub slow_record {
    return {
        id => $_[0],
        name => $_[1],
        age => $_[2],
        email => $_[3],
    };
}

# FAST: Array with constants for indices
use constant {
    ID => 0,
    NAME => 1,
    AGE => 2,
    EMAIL => 3,
};

sub fast_record {
    return [@_];
}

# Access: $record->[NAME] instead of $record->{name}
```

## Memory Optimization

### Reducing Memory Usage

```perl
# MEMORY INTENSIVE: Slurping large files
sub memory_intensive {
    open my $fh, '<', 'huge_file.txt' or die $!;
    my @lines = <$fh>;  # Loads entire file into memory
    close $fh;

    for my $line (@lines) {
        process_line($line);
    }
}

# MEMORY EFFICIENT: Line-by-line processing
sub memory_efficient {
    open my $fh, '<', 'huge_file.txt' or die $!;

    while (my $line = <$fh>) {
        process_line($line);
    }

    close $fh;
}

# MEMORY INTENSIVE: Building large structures
sub build_large_hash {
    my %data;

    for my $i (1..1_000_000) {
        $data{$i} = {
            id => $i,
            value => rand(),
            timestamp => time(),
            metadata => { foo => 'bar' },
        };
    }

    return \%data;
}

# MEMORY EFFICIENT: Using packed data
sub build_packed_data {
    my $packed = "";

    for my $i (1..1_000_000) {
        # Pack: unsigned int, double, unsigned int
        $packed .= pack("NdN", $i, rand(), time());
    }

    return \$packed;
}

# Retrieve packed data
sub get_packed_record {
    my ($packed_ref, $index) = @_;
    my $offset = $index * 16;  # Each record is 16 bytes

    my ($id, $value, $timestamp) = unpack("NdN",
        substr($$packed_ref, $offset, 16)
    );

    return { id => $id, value => $value, timestamp => $timestamp };
}
```

### Circular References and Memory Leaks

```perl
# MEMORY LEAK: Circular reference
sub create_leak {
    my $node1 = { name => 'Node1' };
    my $node2 = { name => 'Node2' };

    $node1->{next} = $node2;
    $node2->{prev} = $node1;  # Circular reference!

    # Memory not freed when variables go out of scope
}

# SOLUTION 1: Weak references
use Scalar::Util qw(weaken);

sub no_leak_weak {
    my $node1 = { name => 'Node1' };
    my $node2 = { name => 'Node2' };

    $node1->{next} = $node2;
    $node2->{prev} = $node1;
    weaken($node2->{prev});  # Make it a weak reference

    # Memory properly freed
}

# SOLUTION 2: Explicit cleanup
sub no_leak_cleanup {
    my $node1 = { name => 'Node1' };
    my $node2 = { name => 'Node2' };

    $node1->{next} = $node2;
    $node2->{prev} = $node1;

    # Clean up before scope exit
    delete $node2->{prev};
}

# Detect leaks
use Devel::Leak;

my $handle;
my $count = Devel::Leak::NoteSV($handle);

# Code that might leak
create_leak() for 1..100;

my $new_count = Devel::Leak::CheckSV($handle);
say "Leaked " . ($new_count - $count) . " SVs";
```

## Algorithm Optimization

### Choosing Better Algorithms

```perl
# O(n²) - SLOW for large datasets
sub find_duplicates_slow {
    my ($array) = @_;
    my @duplicates;

    for (my $i = 0; $i < @$array; $i++) {
        for (my $j = $i + 1; $j < @$array; $j++) {
            if ($array->[$i] eq $array->[$j]) {
                push @duplicates, $array->[$i];
                last;
            }
        }
    }

    return \@duplicates;
}

# O(n) - FAST using hash
sub find_duplicates_fast {
    my ($array) = @_;
    my (%seen, @duplicates);

    for my $item (@$array) {
        push @duplicates, $item if $seen{$item}++;
    }

    return \@duplicates;
}

# Caching/Memoization
use Memoize;

sub expensive_calculation {
    my ($n) = @_;
    sleep(1);  # Simulate expensive operation
    return $n * $n;
}

memoize('expensive_calculation');

# First call: slow
my $result1 = expensive_calculation(42);  # Takes 1 second

# Subsequent calls: instant
my $result2 = expensive_calculation(42);  # Returns cached result

# Manual memoization with limits
{
    my %cache;
    my $max_cache_size = 100;

    sub cached_function {
        my ($key) = @_;

        # Check cache
        return $cache{$key} if exists $cache{$key};

        # Compute result
        my $result = expensive_computation($key);

        # Limit cache size
        if (keys %cache >= $max_cache_size) {
            # Remove oldest entry (simple FIFO)
            my $oldest = (sort keys %cache)[0];
            delete $cache{$oldest};
        }

        # Cache and return
        return $cache{$key} = $result;
    }
}
```

### Lazy Evaluation

```perl
# EAGER: Computes everything upfront
sub eager_processing {
    my ($data) = @_;

    my @results = map { expensive_transform($_) } @$data;
    return \@results;
}

# LAZY: Computes only when needed
sub lazy_processing {
    my ($data) = @_;

    return sub {
        my ($index) = @_;
        return unless $index < @$data;

        state %cache;
        return $cache{$index} //= expensive_transform($data->[$index]);
    };
}

# Usage
my $lazy = lazy_processing(\@huge_dataset);
my $item5 = $lazy->(5);  # Only computes the 5th item

# Iterator pattern for large datasets
sub make_iterator {
    my ($data) = @_;
    my $index = 0;

    return sub {
        return unless $index < @$data;
        return $data->[$index++];
    };
}

my $iter = make_iterator(\@data);
while (my $item = $iter->()) {
    process($item);
    last if $processed++ > 100;  # Can stop early
}
```

## XS and Inline::C

### Using Inline::C for Performance

```perl
use Inline C => <<'END_C';
    int fast_sum(SV* array_ref) {
        AV* array = (AV*)SvRV(array_ref);
        int len = av_len(array) + 1;
        int sum = 0;

        for (int i = 0; i < len; i++) {
            SV** elem = av_fetch(array, i, 0);
            if (elem && SvIOK(*elem)) {
                sum += SvIV(*elem);
            }
        }

        return sum;
    }

    void fast_sort(SV* array_ref) {
        AV* array = (AV*)SvRV(array_ref);
        int len = av_len(array) + 1;

        // Simple bubble sort for demonstration
        for (int i = 0; i < len - 1; i++) {
            for (int j = 0; j < len - i - 1; j++) {
                SV** elem1 = av_fetch(array, j, 0);
                SV** elem2 = av_fetch(array, j + 1, 0);

                if (elem1 && elem2 && SvIV(*elem1) > SvIV(*elem2)) {
                    SV* temp = *elem1;
                    av_store(array, j, *elem2);
                    av_store(array, j + 1, temp);
                }
            }
        }
    }
END_C

# Use the C functions
my @numbers = map { int(rand(1000)) } 1..10000;
my $sum = fast_sum(\@numbers);
fast_sort(\@numbers);
```

## Database Optimization

### Efficient Database Operations

```perl
# SLOW: Individual queries
sub slow_db_insert {
    my ($dbh, $data) = @_;

    my $sth = $dbh->prepare("INSERT INTO users (name, email) VALUES (?, ?)");

    for my $user (@$data) {
        $sth->execute($user->{name}, $user->{email});
    }
}

# FAST: Bulk insert
sub fast_db_insert {
    my ($dbh, $data) = @_;

    $dbh->begin_work;

    my $sth = $dbh->prepare("INSERT INTO users (name, email) VALUES (?, ?)");

    for my $user (@$data) {
        $sth->execute($user->{name}, $user->{email});
    }

    $dbh->commit;
}

# FASTER: Single multi-row insert
sub faster_db_insert {
    my ($dbh, $data) = @_;

    return unless @$data;

    my @placeholders = map { "(?, ?)" } @$data;
    my $sql = "INSERT INTO users (name, email) VALUES " .
              join(", ", @placeholders);

    my @values = map { $_->{name}, $_->{email} } @$data;

    $dbh->do($sql, undef, @values);
}

# Use prepared statement caching
sub cached_query {
    my ($dbh, $id) = @_;

    # prepare_cached reuses statement handles
    my $sth = $dbh->prepare_cached(
        "SELECT * FROM users WHERE id = ?"
    );

    $sth->execute($id);
    return $sth->fetchrow_hashref;
}
```

## Real-World Optimization Example

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Benchmark qw(cmpthese);

# Log parser optimization
package LogParser;

# Version 1: Naive implementation
sub parse_v1 {
    my ($file) = @_;
    my @results;

    open my $fh, '<', $file or die $!;

    while (my $line = <$fh>) {
        chomp $line;

        # Inefficient: multiple regex matches
        if ($line =~ /^\[(\d{4}-\d{2}-\d{2})/) {
            my $date = $1;

            if ($line =~ /ERROR/) {
                my $entry = { date => $date, level => 'ERROR' };

                if ($line =~ /user:\s*(\w+)/) {
                    $entry->{user} = $1;
                }

                if ($line =~ /message:\s*(.+)$/) {
                    $entry->{message} = $1;
                }

                push @results, $entry;
            }
        }
    }

    close $fh;
    return \@results;
}

# Version 2: Optimized
sub parse_v2 {
    my ($file) = @_;
    my @results;

    # Pre-compile regexes
    my $line_regex = qr/^\[(\d{4}-\d{2}-\d{2})[^\]]*\]\s+(\w+)\s+user:\s*(\w+)\s+message:\s*(.+)$/;
    my $error_check = qr/ERROR/;

    open my $fh, '<', $file or die $!;

    while (my $line = <$fh>) {
        # Skip non-error lines early
        next unless index($line, 'ERROR') >= 0;

        # Single regex to capture everything
        if ($line =~ $line_regex) {
            push @results, {
                date => $1,
                level => $2,
                user => $3,
                message => $4,
            } if $2 eq 'ERROR';
        }
    }

    close $fh;
    return \@results;
}

# Version 3: Memory-mapped for huge files
use File::Map qw(map_file);

sub parse_v3 {
    my ($file) = @_;
    my @results;

    map_file my $content, $file;

    # Process in chunks
    my $regex = qr/^\[(\d{4}-\d{2}-\d{2})[^\]]*\]\s+ERROR\s+user:\s*(\w+)\s+message:\s*(.+)$/m;

    while ($content =~ /$regex/g) {
        push @results, {
            date => $1,
            level => 'ERROR',
            user => $2,
            message => $3,
        };
    }

    return \@results;
}

# Benchmark
cmpthese(-3, {
    'v1_naive' => sub { parse_v1('test.log') },
    'v2_optimized' => sub { parse_v2('test.log') },
    'v3_mmap' => sub { parse_v3('test.log') },
});
```

## Performance Best Practices

1. **Profile first** - Don't guess where the bottlenecks are
2. **Optimize algorithms before code** - O(n) beats optimized O(n²)
3. **Cache expensive operations** - But watch memory usage
4. **Pre-compile regexes** - Especially in loops
5. **Use built-in functions** - They're optimized C code
6. **Avoid premature optimization** - Clean code first, fast code second
7. **Benchmark alternatives** - Measure, don't assume
8. **Consider memory vs speed tradeoffs** - Sometimes caching hurts
9. **Use appropriate data structures** - Hashes for lookups, arrays for order
10. **Know when to use XS/C** - For truly performance-critical code

## Optimization Checklist

Before optimizing:
- [ ] Is the code correct?
- [ ] Is the code readable?
- [ ] Have you profiled it?
- [ ] Is this the actual bottleneck?

While optimizing:
- [ ] Benchmark before and after
- [ ] Test that behavior hasn't changed
- [ ] Document why the optimization is needed
- [ ] Consider maintenance cost

After optimizing:
- [ ] Is the improvement significant?
- [ ] Is the code still maintainable?
- [ ] Are the tests still passing?
- [ ] Have you updated documentation?

## Conclusion

Performance optimization in Perl is about understanding your bottlenecks and applying the right techniques. Most performance problems can be solved with better algorithms, caching, or pre-compilation. Only resort to XS or Inline::C when you've exhausted Perl-level optimizations.

Remember: the fastest code is code that doesn't run. Before optimizing, ask if you can eliminate work entirely. And always, always profile first.

---

*Next: Building command-line tools. We'll create professional CLI applications with argument parsing, help text, and user-friendly interfaces.*