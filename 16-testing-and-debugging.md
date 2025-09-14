# Chapter 16: Testing and Debugging

> "Testing can show the presence of bugs, but never their absence. Debugging is twice as hard as writing code. Therefore, if you write code as cleverly as possible, you are, by definition, not smart enough to debug it." - Brian Kernighan (paraphrased)

Perl has one of the strongest testing cultures in programming. Every CPAN module comes with tests, and the Perl community takes testing seriously. This chapter covers everything from basic unit tests to advanced debugging techniques, helping you write code that works and stays working.

## Testing Fundamentals

### Basic Testing with Test::More

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use Test::More tests => 10;  # Declare number of tests

# Basic assertions
ok(1 + 1 == 2, 'Basic math works');
ok(-f 'script.pl', 'Script file exists');

# Equality tests
is(2 + 2, 4, 'Addition works');
is(uc('hello'), 'HELLO', 'uc() works correctly');

# String comparison
isnt('foo', 'bar', 'foo is not bar');

# Pattern matching
like('Hello World', qr/World/, 'String contains World');
unlike('Hello World', qr/Goodbye/, 'String does not contain Goodbye');

# Comparison operators
cmp_ok(5, '>', 3, '5 is greater than 3');
cmp_ok('abc', 'lt', 'def', 'String comparison works');

# Complex data structures
is_deeply(
    [1, 2, 3],
    [1, 2, 3],
    'Arrays match'
);

# Or use done_testing() for dynamic test counts
use Test::More;

ok(1, 'Test 1');
ok(1, 'Test 2');

done_testing();  # Automatically counts tests
```

### Testing Modules

```perl
# t/00-load.t
use Test::More tests => 3;

BEGIN {
    use_ok('My::Module');
    use_ok('My::Module::Helper');
    use_ok('My::Module::Utils');
}

diag("Testing My::Module $My::Module::VERSION, Perl $], $^X");

# t/01-basic.t
use Test::More;
use My::Module;

# Test object creation
my $obj = My::Module->new(name => 'test');
isa_ok($obj, 'My::Module', 'Object created correctly');

# Test methods
can_ok($obj, qw(process validate save load));

# Test basic functionality
is($obj->name, 'test', 'Name accessor works');

$obj->process('input');
is($obj->status, 'processed', 'Processing sets correct status');

done_testing();
```

### Test-Driven Development (TDD)

```perl
# Write tests first
use Test::More;
use Calculator;

my $calc = Calculator->new();
isa_ok($calc, 'Calculator');

# Test addition
is($calc->add(2, 3), 5, '2 + 3 = 5');
is($calc->add(-1, 1), 0, '-1 + 1 = 0');
is($calc->add(0, 0), 0, '0 + 0 = 0');

# Test subtraction
is($calc->subtract(5, 3), 2, '5 - 3 = 2');
is($calc->subtract(0, 5), -5, '0 - 5 = -5');

# Test division
is($calc->divide(10, 2), 5, '10 / 2 = 5');

# Test error handling
eval { $calc->divide(10, 0) };
like($@, qr/Division by zero/, 'Division by zero throws error');

done_testing();

# Then implement the module
package Calculator;
use Carp;

sub new {
    my $class = shift;
    return bless {}, $class;
}

sub add {
    my ($self, $a, $b) = @_;
    return $a + $b;
}

sub subtract {
    my ($self, $a, $b) = @_;
    return $a - $b;
}

sub divide {
    my ($self, $a, $b) = @_;
    croak "Division by zero" if $b == 0;
    return $a / $b;
}

1;
```

## Advanced Testing

### Test::Deep for Complex Structures

```perl
use Test::More;
use Test::Deep;

my $got = {
    name => 'Alice',
    age => 30,
    skills => ['perl', 'python', 'ruby'],
    address => {
        street => '123 Main St',
        city => 'New York',
        zip => '10001',
    },
    metadata => {
        created => '2024-01-15T10:30:00',
        modified => '2024-01-16T14:45:00',
    },
};

cmp_deeply($got, {
    name => 'Alice',
    age => code(sub { $_[0] >= 18 && $_[0] <= 100 }),
    skills => bag('perl', 'python', 'ruby'),  # Order doesn't matter
    address => superhashof({
        city => 'New York',  # Must have city, other keys optional
    }),
    metadata => {
        created => re(qr/^\d{4}-\d{2}-\d{2}/),
        modified => ignore(),  # Don't care about this value
    },
}, 'Structure matches expectations');

# Array testing
my @data = (1, 2, 3, { id => 'abc123' }, undef);

cmp_deeply(\@data, [
    1, 2, 3,
    { id => re(qr/^[a-z]+\d+$/) },
    undef,
], 'Array with mixed types');

# Set comparison
cmp_deeply(
    [1, 2, 3, 3, 2, 1],
    set(1, 2, 3),
    'Contains exactly these values (duplicates ignored)'
);
```

### Mocking and Test Doubles

```perl
use Test::More;
use Test::MockModule;
use Test::MockObject;

# Mock a module
my $mock = Test::MockModule->new('LWP::UserAgent');
$mock->mock('get', sub {
    my ($self, $url) = @_;

    # Return fake response
    my $response = Test::MockObject->new();
    $response->set_true('is_success');
    $response->set_always('content', '{"status":"ok"}');

    return $response;
});

# Now LWP::UserAgent->get returns our mock
use LWP::UserAgent;
my $ua = LWP::UserAgent->new();
my $resp = $ua->get('http://example.com');

ok($resp->is_success, 'Mock response is successful');
is($resp->content, '{"status":"ok"}', 'Mock content correct');

# Mock database handle
my $mock_dbh = Test::MockObject->new();
$mock_dbh->set_always('prepare', $mock_dbh);
$mock_dbh->set_always('execute', 1);
$mock_dbh->set_series('fetchrow_array',
    ['Alice', 30],
    ['Bob', 25],
    undef,  # End of results
);

# Test code that uses database
my $sth = $mock_dbh->prepare("SELECT name, age FROM users");
$sth->execute();

my @users;
while (my @row = $sth->fetchrow_array()) {
    push @users, { name => $row[0], age => $row[1] };
}

is(scalar @users, 2, 'Got two users');
is($users[0]{name}, 'Alice', 'First user is Alice');
```

### Testing Exceptions

```perl
use Test::More;
use Test::Exception;
use Test::Fatal;

# Test::Exception
throws_ok {
    die "Something went wrong";
} qr/went wrong/, 'Dies with expected message';

dies_ok {
    risky_operation();
} 'risky_operation dies';

lives_ok {
    safe_operation();
} 'safe_operation lives';

# Test::Fatal (more modern)
use Test::Fatal;

my $exception = exception {
    die "Oops!";
};

like($exception, qr/Oops/, 'Got expected exception');

# Test specific exception classes
{
    package MyException;
    use overload '""' => sub { shift->{message} };

    sub new {
        my ($class, $message) = @_;
        return bless { message => $message }, $class;
    }
}

my $error = exception {
    die MyException->new("Custom error");
};

isa_ok($error, 'MyException');
is($error->{message}, 'Custom error', 'Exception has correct message');
```

## Debugging Techniques

### Basic Debugging

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use Data::Dumper;

# Print debugging
my $data = { foo => 'bar', baz => [1, 2, 3] };
print "Debug: ", Dumper($data);

# Conditional debugging
my $DEBUG = $ENV{DEBUG} || 0;
print "Processing data...\n" if $DEBUG;

# Better: use a debug function
sub debug {
    return unless $ENV{DEBUG};
    my $msg = shift;
    my ($package, $filename, $line) = caller;
    print STDERR "[$package:$line] $msg\n";
}

debug("Starting process");

# Smart::Comments for easy debugging
use Smart::Comments;

### $data
my $result = complex_calculation();
### Result: $result

# Assertions
### check: $result > 0
### assert: defined $data->{id}

# Progress bars
for my $i (0..100) {  ### Processing [===         ] % done
    # Do work
    sleep(0.01);
}
```

### The Perl Debugger

```perl
# Run with debugger
# perl -d script.pl

# Common debugger commands:
# h         - help
# l         - list code
# n         - next line (step over)
# s         - step into
# c         - continue
# b 42      - set breakpoint at line 42
# b subname - set breakpoint at subroutine
# p $var    - print variable
# x $ref    - dump reference
# w         - where (stack trace)
# q         - quit

# Add breakpoint in code
$DB::single = 1;  # Debugger stops here

# Conditional breakpoint
$DB::single = 1 if $count > 100;

# Interactive debugging session example
sub process_data {
    my ($data) = @_;

    $DB::single = 1;  # Stop here in debugger

    for my $item (@$data) {
        my $result = transform($item);
        validate($result);
    }
}
```

### Devel::NYTProf for Profiling

```perl
# Profile your code
# perl -d:NYTProf script.pl
# nytprofhtml
# open nytprof/index.html

# Or programmatically
use Devel::NYTProf;

DB::enable_profile();

# Code to profile
expensive_operation();

DB::disable_profile();
DB::finish_profile();

# Analyze results
system('nytprofhtml');
system('open nytprof/index.html');

# Example of code that needs profiling
sub slow_function {
    my @results;

    for my $i (1..10000) {
        # Inefficient string concatenation
        my $str = "";
        for my $j (1..100) {
            $str .= "x";  # Bad: creates new string each time
        }
        push @results, $str;
    }

    return \@results;
}

# Better version after profiling
sub fast_function {
    my @results;

    for my $i (1..10000) {
        # Efficient: use x operator
        push @results, "x" x 100;
    }

    return \@results;
}
```

## Test Organization

### Test Suite Structure

```perl
# Project structure
# MyApp/
# ├── lib/
# │   └── MyApp/
# │       ├── Model.pm
# │       ├── View.pm
# │       └── Controller.pm
# ├── t/
# │   ├── 00-load.t
# │   ├── 01-unit/
# │   │   ├── model.t
# │   │   ├── view.t
# │   │   └── controller.t
# │   ├── 02-integration/
# │   │   └── api.t
# │   ├── 03-acceptance/
# │   │   └── user_stories.t
# │   └── lib/
# │       └── Test/
# │           └── MyApp.pm

# t/lib/Test/MyApp.pm - Shared test utilities
package Test::MyApp;
use strict;
use warnings;
use Test::More;
use File::Temp;
use base 'Exporter';

our @EXPORT = qw(
    create_test_db
    create_test_user
    cleanup_test_data
);

sub create_test_db {
    my $tmpdir = File::Temp->newdir();
    my $dbfile = "$tmpdir/test.db";

    # Create and populate test database
    my $dbh = DBI->connect("dbi:SQLite:dbname=$dbfile");
    $dbh->do($_) for read_schema();

    return $dbh;
}

sub create_test_user {
    my %args = @_;
    return {
        id => $args{id} // 1,
        name => $args{name} // 'Test User',
        email => $args{email} // 'test@example.com',
    };
}

sub cleanup_test_data {
    # Clean up any test files, databases, etc.
}

1;
```

### Test Coverage

```perl
# Check test coverage
# cover -test

# Or manually:
# perl -MDevel::Cover script.pl
# cover

# Configure coverage
# .coverrc file
coverage_class = Devel::Cover
db             = cover_db
ignore         = t/
                 /usr/
select         = lib/
outputdir      = coverage_report

# Add coverage badge to README
# cpanm Devel::Cover::Report::Coveralls
# cover -report coveralls

# Example of improving coverage
package MyModule;

sub process {
    my ($self, $input) = @_;

    # Branch 1
    if (!defined $input) {
        return undef;
    }

    # Branch 2
    if ($input eq '') {
        return '';
    }

    # Branch 3
    if ($input =~ /^\d+$/) {
        return $input * 2;
    }

    # Branch 4
    return uc($input);
}

# Test file ensuring 100% coverage
use Test::More;
use MyModule;

my $obj = MyModule->new();

# Test all branches
is($obj->process(undef), undef, 'Handles undef');
is($obj->process(''), '', 'Handles empty string');
is($obj->process('42'), 84, 'Handles numbers');
is($obj->process('hello'), 'HELLO', 'Handles strings');

done_testing();
```

## Continuous Integration Testing

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        perl-version: ['5.32', '5.34', '5.36']

    steps:
    - uses: actions/checkout@v2

    - name: Setup Perl
      uses: shogo82148/actions-setup-perl@v1
      with:
        perl-version: ${{ matrix.perl-version }}

    - name: Install dependencies
      run: |
        cpanm --installdeps --notest .
        cpanm Test::More Test::Deep Test::Exception

    - name: Run tests
      run: prove -lv t/

    - name: Check coverage
      run: |
        cpanm Devel::Cover
        cover -test -report coveralls
```

### Test Automation Script

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use TAP::Harness;
use File::Find;
use Getopt::Long;

my ($verbose, $coverage, $parallel) = (0, 0, 1);
GetOptions(
    'verbose|v' => \$verbose,
    'coverage|c' => \$coverage,
    'jobs|j=i' => \$parallel,
);

# Find all test files
my @tests;
find(sub {
    push @tests, $File::Find::name if /\.t$/;
}, 't/');

@tests = sort @tests;

# Run with coverage if requested
if ($coverage) {
    $ENV{HARNESS_PERL_SWITCHES} = '-MDevel::Cover';
}

# Run tests
my $harness = TAP::Harness->new({
    verbosity => $verbose,
    jobs => $parallel,
    color => 1,
    lib => ['lib'],
});

my $aggregator = $harness->runtests(@tests);

# Generate coverage report
if ($coverage) {
    system('cover');
    say "\nCoverage report generated in cover_db/";
    say "Run 'cover -report html' for HTML report";
}

# Exit with appropriate status
exit($aggregator->all_passed() ? 0 : 1);
```

## Real-World Testing Example

```perl
#!/usr/bin/env perl
# t/web_scraper.t
use Modern::Perl '2023';
use Test::More;
use Test::MockModule;
use Test::Deep;
use Test::Exception;

# Module we're testing
use lib 'lib';
use WebScraper;

# Mock HTTP responses
my $mock_ua = Test::MockModule->new('LWP::UserAgent');
my @responses;

$mock_ua->mock('get', sub {
    my ($self, $url) = @_;
    my $response = shift @responses;

    unless ($response) {
        my $mock = Test::MockObject->new();
        $mock->set_false('is_success');
        $mock->set_always('status_line', '404 Not Found');
        return $mock;
    }

    return $response;
});

# Test basic scraping
subtest 'Basic scraping' => sub {
    my $html = <<'HTML';
    <html>
        <head><title>Test Page</title></head>
        <body>
            <h1>Welcome</h1>
            <ul class="items">
                <li>Item 1</li>
                <li>Item 2</li>
                <li>Item 3</li>
            </ul>
        </body>
    </html>
HTML

    my $mock_response = Test::MockObject->new();
    $mock_response->set_true('is_success');
    $mock_response->set_always('decoded_content', $html);

    @responses = ($mock_response);

    my $scraper = WebScraper->new(url => 'http://example.com');
    my $data = $scraper->scrape();

    is($data->{title}, 'Test Page', 'Extracted title');
    is($data->{heading}, 'Welcome', 'Extracted heading');
    cmp_deeply($data->{items}, ['Item 1', 'Item 2', 'Item 3'], 'Extracted items');
};

# Test error handling
subtest 'Error handling' => sub {
    @responses = ();  # No responses = 404

    my $scraper = WebScraper->new(url => 'http://example.com');

    throws_ok {
        $scraper->scrape();
    } qr/Failed to fetch/, 'Throws on HTTP error';
};

# Test retry logic
subtest 'Retry logic' => sub {
    my $fail_response = Test::MockObject->new();
    $fail_response->set_false('is_success');
    $fail_response->set_always('status_line', '500 Server Error');

    my $success_response = Test::MockObject->new();
    $success_response->set_true('is_success');
    $success_response->set_always('decoded_content', '<html><title>OK</title></html>');

    @responses = ($fail_response, $fail_response, $success_response);

    my $scraper = WebScraper->new(
        url => 'http://example.com',
        max_retries => 3,
    );

    my $data;
    lives_ok {
        $data = $scraper->scrape();
    } 'Succeeds after retries';

    is($data->{title}, 'OK', 'Got data after retries');
};

# Test rate limiting
subtest 'Rate limiting' => sub {
    my $scraper = WebScraper->new(
        url => 'http://example.com',
        rate_limit => 2,  # 2 requests per second
    );

    # Mock successful responses
    for (1..5) {
        my $mock = Test::MockObject->new();
        $mock->set_true('is_success');
        $mock->set_always('decoded_content', '<html></html>');
        push @responses, $mock;
    }

    my $start = time();

    for (1..5) {
        $scraper->scrape();
    }

    my $elapsed = time() - $start;
    cmp_ok($elapsed, '>=', 2, 'Rate limiting enforced');
};

done_testing();
```

## Best Practices

1. **Write tests first** - TDD helps design better APIs
2. **Test at multiple levels** - Unit, integration, acceptance
3. **Keep tests fast** - Mock external dependencies
4. **Test edge cases** - Empty input, undef, large data
5. **Use descriptive test names** - They document behavior
6. **Maintain test data** - Use fixtures and factories
7. **Run tests frequently** - Before every commit
8. **Measure coverage** - But don't obsess over 100%
9. **Test error conditions** - Not just happy paths
10. **Keep tests maintainable** - Refactor test code too

## Conclusion

Testing and debugging are essential skills for any Perl programmer. Perl's testing ecosystem is mature and comprehensive, providing tools for every testing need. Good tests give you confidence to refactor, upgrade, and maintain your code over time.

Remember: untested code is broken code. It might work today, but without tests, you can't be sure it will work tomorrow.

---

*Next: Performance and optimization. We'll explore how to make your Perl code run faster and use less memory.*