# Chapter 14: CPAN - The Treasure Trove

> "CPAN is the language. Perl is just its syntax." - chromatic

The Comprehensive Perl Archive Network (CPAN) is Perl's killer feature. With over 200,000 modules solving virtually every programming problem imaginable, CPAN transforms Perl from a language into an ecosystem. This chapter shows you how to navigate this treasure trove, evaluate modules, contribute your own, and leverage CPAN to write less code and solve more problems.

## Understanding CPAN

### The CPAN Ecosystem

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# CPAN consists of:
# 1. The archive - mirrors worldwide containing modules
# 2. MetaCPAN - modern web interface for searching and browsing
# 3. CPAN clients - tools to install modules (cpan, cpanm, carton)
# 4. PAUSE - Perl Authors Upload Server for contributors
# 5. CPAN Testers - automated testing across platforms

# Finding modules:
# - MetaCPAN: https://metacpan.org
# - Search by task, author, or module name
# - Check ratings, test results, and dependencies
# - Read documentation and reviews

# Understanding module naming:
# Foo::Bar       - Hierarchical namespace
# Foo::Bar::XS   - XS (C) implementation
# Foo::Bar::PP   - Pure Perl implementation
# Foo::Bar::Tiny - Minimal version
# Foo::Bar::Mo   - Modern/Moose-based version
```

### Installing Modules

```perl
# Using cpanminus (recommended)
# Install: curl -L https://cpanmin.us | perl - App::cpanminus

# Install a module
system('cpanm Module::Name');

# Install specific version
system('cpanm Module::Name@1.23');

# Install from GitHub
system('cpanm git://github.com/user/repo.git');

# Install with dependencies
system('cpanm --installdeps .');

# Install without tests (faster but risky)
system('cpanm --notest Module::Name');

# Local installation
system('cpanm -l ~/perl5 Module::Name');

# Using traditional CPAN client
use CPAN;
CPAN::Shell->install('Module::Name');

# Or from command line:
# cpan Module::Name
```

### Managing Dependencies with Carton

```perl
# Carton is like Bundler for Ruby or npm for Node.js

# Create cpanfile
my $cpanfile = <<'END';
requires 'Plack', '1.0047';
requires 'DBI', '1.643';
requires 'DBD::SQLite', '1.66';
requires 'Mojo', '9.0';

on 'test' => sub {
    requires 'Test::More', '1.302183';
    requires 'Test::Exception';
    requires 'Test::MockObject';
};

on 'develop' => sub {
    requires 'Perl::Tidy';
    requires 'Perl::Critic';
    requires 'Devel::NYTProf';
};

feature 'postgres', 'PostgreSQL support' => sub {
    requires 'DBD::Pg';
};
END

# Install dependencies
system('carton install');

# Run with local dependencies
system('carton exec perl script.pl');

# Deploy with dependencies
system('carton bundle');
```

## Essential CPAN Modules

### Web Development

```perl
# Mojolicious - Real-time web framework
use Mojolicious::Lite;

get '/' => sub {
    my $c = shift;
    $c->render(text => 'Hello World!');
};

app->start;

# Plack - PSGI toolkit
use Plack::Builder;

my $app = sub {
    my $env = shift;
    return [200, ['Content-Type' => 'text/plain'], ['Hello World']];
};

builder {
    enable 'Debug';
    enable 'Session';
    $app;
};

# Dancer2 - Lightweight web framework
use Dancer2;

get '/' => sub {
    return 'Hello World';
};

dance;
```

### Database Access

```perl
# DBIx::Class - ORM
package MyApp::Schema::Result::User;
use base 'DBIx::Class::Core';

__PACKAGE__->table('users');
__PACKAGE__->add_columns(qw/id name email/);
__PACKAGE__->set_primary_key('id');
__PACKAGE__->has_many(posts => 'MyApp::Schema::Result::Post', 'user_id');

# Using DBIx::Class
my $schema = MyApp::Schema->connect($dsn, $user, $pass);
my $users = $schema->resultset('User')->search({
    created => { '>' => '2024-01-01' }
});

# SQL::Abstract - Generate SQL from Perl data structures
use SQL::Abstract;

my $sql = SQL::Abstract->new;
my ($stmt, @bind) = $sql->select(
    'users',
    ['name', 'email'],
    {
        status => 'active',
        age => { '>' => 18 },
    }
);
```

### Testing

```perl
# Test::More - Standard testing
use Test::More tests => 3;

ok(1 + 1 == 2, 'Math works');
is(uc('hello'), 'HELLO', 'uc works');
like('Hello World', qr/World/, 'Pattern matches');

# Test::Deep - Deep structure comparison
use Test::Deep;

cmp_deeply(
    $got,
    {
        name => 'Alice',
        tags => bag(qw/perl programming/),
        meta => superhashof({
            created => re(qr/^\d{4}-\d{2}-\d{2}$/),
        }),
    },
    'Structure matches'
);

# Test::MockObject - Mock objects for testing
use Test::MockObject;

my $mock = Test::MockObject->new();
$mock->set_always('fetch_data', { id => 1, name => 'Test' });
$mock->set_true('save');

ok($mock->save(), 'Save returns true');
```

### Date and Time

```perl
# DateTime - Comprehensive date/time handling
use DateTime;

my $dt = DateTime->now(time_zone => 'America/New_York');
$dt->add(days => 7, hours => 3);
say $dt->strftime('%Y-%m-%d %H:%M:%S');

# Time::Piece - Core module for simple date/time
use Time::Piece;

my $t = localtime;
say $t->datetime;  # ISO 8601
say $t->epoch;
say $t->day_of_week;

my $birthday = Time::Piece->strptime('1990-01-15', '%Y-%m-%d');
my $age = ($t - $birthday)->years;
```

## Creating Your Own CPAN Module

### Module Structure

```perl
# Directory structure:
# My-Module/
# ├── lib/
# │   └── My/
# │       └── Module.pm
# ├── t/
# │   ├── 00-load.t
# │   └── 01-basic.t
# ├── Makefile.PL
# ├── MANIFEST
# ├── README.md
# ├── Changes
# └── LICENSE

# lib/My/Module.pm
package My::Module;
use strict;
use warnings;
use feature 'signatures';
no warnings 'experimental::signatures';

our $VERSION = '0.01';

sub new($class, %args) {
    return bless \%args, $class;
}

sub do_something($self) {
    return "Doing something!";
}

1;

__END__

=head1 NAME

My::Module - A brief description

=head1 SYNOPSIS

    use My::Module;

    my $obj = My::Module->new();
    $obj->do_something();

=head1 DESCRIPTION

This module provides...

=head1 METHODS

=head2 new

Constructor. Creates a new My::Module object.

=head2 do_something

Does something useful.

=head1 AUTHOR

Your Name <your.email@example.com>

=head1 LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

=cut
```

### Build Files

```perl
# Makefile.PL (ExtUtils::MakeMaker)
use ExtUtils::MakeMaker;

WriteMakefile(
    NAME             => 'My::Module',
    AUTHOR           => 'Your Name <email@example.com>',
    VERSION_FROM     => 'lib/My/Module.pm',
    ABSTRACT_FROM    => 'lib/My/Module.pm',
    LICENSE          => 'perl_5',
    MIN_PERL_VERSION => '5.016',
    CONFIGURE_REQUIRES => {
        'ExtUtils::MakeMaker' => '0',
    },
    TEST_REQUIRES => {
        'Test::More' => '0',
    },
    PREREQ_PM => {
        'strict'   => 0,
        'warnings' => 0,
    },
    META_MERGE => {
        'meta-spec' => { version => 2 },
        resources => {
            repository => {
                type => 'git',
                url  => 'https://github.com/user/My-Module.git',
                web  => 'https://github.com/user/My-Module',
            },
        },
    },
    dist  => { COMPRESS => 'gzip -9f', SUFFIX => 'gz', },
    clean => { FILES => 'My-Module-*' },
);

# Or Build.PL (Module::Build)
use Module::Build;

my $builder = Module::Build->new(
    module_name         => 'My::Module',
    license             => 'perl',
    dist_author         => 'Your Name <email@example.com>',
    dist_version_from   => 'lib/My/Module.pm',
    requires => {
        'perl'     => '5.016',
        'strict'   => 0,
        'warnings' => 0,
    },
    test_requires => {
        'Test::More' => 0,
    },
    add_to_cleanup      => [ 'My-Module-*' ],
    meta_merge => {
        resources => {
            repository => 'https://github.com/user/My-Module',
        },
    },
);

$builder->create_build_script();
```

### Testing Your Module

```perl
# t/00-load.t
use Test::More tests => 1;

BEGIN {
    use_ok('My::Module') || print "Bail out!\n";
}

diag("Testing My::Module $My::Module::VERSION, Perl $], $^X");

# t/01-basic.t
use Test::More;
use My::Module;

my $obj = My::Module->new(name => 'test');
isa_ok($obj, 'My::Module');

is($obj->do_something(), 'Doing something!', 'do_something works');

done_testing();

# t/02-advanced.t
use Test::More;
use Test::Exception;
use Test::Warnings;

use My::Module;

# Test error handling
throws_ok {
    My::Module->new()->invalid_method();
} qr/Can't locate object method/, 'Dies on invalid method';

# Test edge cases
my $obj = My::Module->new();
is($obj->process(undef), '', 'Handles undef gracefully');
is($obj->process(''), '', 'Handles empty string');

# Test with mock data
my $mock_data = {
    users => [
        { id => 1, name => 'Alice' },
        { id => 2, name => 'Bob' },
    ],
};

my $result = $obj->process_users($mock_data);
is(scalar @{$result->{processed}}, 2, 'Processes all users');

done_testing();
```

## Publishing to CPAN

### Getting a PAUSE Account

```perl
# 1. Register at https://pause.perl.org
# 2. Wait for manual approval (usually 1-2 days)
# 3. You'll receive a PAUSE ID (e.g., YOURNAME)

# Configure cpan-upload
# Install: cpanm CPAN::Uploader

# Create ~/.pause file:
# user YOURNAME
# password your-pause-password

# Or use environment variables:
$ENV{PAUSE_USER} = 'YOURNAME';
$ENV{PAUSE_PASSWORD} = 'your-password';
```

### Preparing for Release

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Pre-release checklist
sub pre_release_checks {
    my $module = shift;

    my @checks = (
        'All tests pass' => sub { system('prove -l t/') == 0 },
        'POD is valid' => sub { system('podchecker lib/') == 0 },
        'MANIFEST is current' => sub { check_manifest() },
        'Version updated' => sub { check_version() },
        'Changes file updated' => sub { check_changes() },
        'No debug code' => sub { !grep_debug_code() },
        'Dependencies correct' => sub { check_dependencies() },
    );

    for (my $i = 0; $i < @checks; $i += 2) {
        my ($desc, $check) = @checks[$i, $i+1];
        if ($check->()) {
            say "✓ $desc";
        } else {
            die "✗ $desc - Fix before release!";
        }
    }
}

sub check_manifest {
    system('perl Makefile.PL');
    system('make manifest');
    my $diff = `diff MANIFEST MANIFEST.bak 2>&1`;
    return $diff eq '';
}

sub check_version {
    open my $fh, '<', 'lib/My/Module.pm' or die $!;
    while (<$fh>) {
        return 1 if /our \$VERSION = '[\d.]+'/;
    }
    return 0;
}

sub check_changes {
    open my $fh, '<', 'Changes' or die $!;
    my $first_line = <$fh>;
    return $first_line =~ /^[\d.]+\s+\d{4}-\d{2}-\d{2}/;
}

sub grep_debug_code {
    my $found = 0;
    system('grep -r "use Data::Dumper" lib/ --exclude-dir=.git') == 0 and $found++;
    system('grep -r "print STDERR" lib/ --exclude-dir=.git') == 0 and $found++;
    system('grep -r "# TODO" lib/ --exclude-dir=.git') == 0 and $found++;
    return $found;
}
```

### Release Process

```perl
# Build distribution
system('perl Makefile.PL');
system('make');
system('make test');
system('make dist');

# This creates My-Module-0.01.tar.gz

# Upload to CPAN
system('cpan-upload My-Module-0.01.tar.gz');

# Or manually:
# 1. Log into https://pause.perl.org
# 2. Upload file through web interface
# 3. Wait for indexing (usually 2-3 hours)

# Monitor your module
# - https://metacpan.org/author/YOURNAME
# - http://www.cpantesters.org/distro/M/My-Module.html
# - https://rt.cpan.org/Dist/Display.html?Name=My-Module
```

## Module Best Practices

### Documentation

```perl
package My::Module;

# POD documentation should include:

=head1 NAME

My::Module - One-line description of module's purpose

=head1 VERSION

Version 0.01

=head1 SYNOPSIS

Quick summary of what the module does.

    use My::Module;

    my $foo = My::Module->new();
    my $result = $foo->do_something();

=head1 DESCRIPTION

A full description of the module and its features.

=head1 METHODS

=head2 new

    my $obj = My::Module->new(%options);

Constructor. Accepts the following options:

=over 4

=item * option1 - Description of option1

=item * option2 - Description of option2

=back

=head2 method_name

    my $result = $obj->method_name($param1, $param2);

Description of what this method does.

=head1 EXAMPLES

    # Example 1: Basic usage
    my $obj = My::Module->new();
    $obj->process($data);

    # Example 2: Advanced usage
    my $obj = My::Module->new(
        verbose => 1,
        timeout => 30,
    );

=head1 DIAGNOSTICS

=over 4

=item C<< Error message here >>

Explanation of error message

=item C<< Another error message >>

Explanation of another error

=back

=head1 CONFIGURATION AND ENVIRONMENT

My::Module requires no configuration files or environment variables.

=head1 DEPENDENCIES

List of modules this module depends on.

=head1 INCOMPATIBILITIES

None reported.

=head1 BUGS AND LIMITATIONS

Please report any bugs or feature requests to...

=head1 SEE ALSO

Links to related modules or resources.

=head1 AUTHOR

Your Name  C<< <email@example.com> >>

=head1 LICENSE AND COPYRIGHT

Copyright (C) 2024 by Your Name

This program is free software; you can redistribute it and/or modify it
under the same terms as Perl itself.

=cut
```

### Versioning

```perl
# Semantic versioning (recommended)
our $VERSION = '1.2.3';  # MAJOR.MINOR.PATCH

# MAJOR - Incompatible API changes
# MINOR - Add functionality (backwards-compatible)
# PATCH - Bug fixes (backwards-compatible)

# Version declaration
package My::Module 0.01;  # Perl 5.12+

# Or traditional
package My::Module;
our $VERSION = '0.01';

# Version checks
use My::Module 1.00;  # Require at least version 1.00

# In code
if ($My::Module::VERSION < 2.00) {
    # Use old API
} else {
    # Use new API
}
```

## Real-World Module: API Client

```perl
package WebService::Example;
use Modern::Perl '2023';
use Moo;
use LWP::UserAgent;
use JSON::XS;
use URI::Escape;
use Carp;

our $VERSION = '0.01';

has 'api_key' => (
    is => 'ro',
    required => 1,
);

has 'base_url' => (
    is => 'ro',
    default => 'https://api.example.com/v1',
);

has 'timeout' => (
    is => 'ro',
    default => 30,
);

has 'ua' => (
    is => 'lazy',
    builder => '_build_ua',
);

sub _build_ua {
    my $self = shift;

    my $ua = LWP::UserAgent->new(
        timeout => $self->timeout,
        agent => "WebService::Example/$VERSION",
    );

    $ua->default_header('X-API-Key' => $self->api_key);
    $ua->default_header('Accept' => 'application/json');

    return $ua;
}

sub get_user {
    my ($self, $user_id) = @_;

    croak "User ID required" unless defined $user_id;

    return $self->_request('GET', "/users/$user_id");
}

sub search_users {
    my ($self, $params) = @_;

    return $self->_request('GET', '/users', $params);
}

sub create_user {
    my ($self, $data) = @_;

    croak "User data required" unless $data && ref $data eq 'HASH';

    return $self->_request('POST', '/users', $data);
}

sub update_user {
    my ($self, $user_id, $data) = @_;

    croak "User ID required" unless defined $user_id;
    croak "Update data required" unless $data && ref $data eq 'HASH';

    return $self->_request('PUT', "/users/$user_id", $data);
}

sub delete_user {
    my ($self, $user_id) = @_;

    croak "User ID required" unless defined $user_id;

    return $self->_request('DELETE', "/users/$user_id");
}

sub _request {
    my ($self, $method, $path, $data) = @_;

    my $url = $self->base_url . $path;

    # Add query params for GET
    if ($method eq 'GET' && $data) {
        my @params;
        for my $key (keys %$data) {
            push @params, uri_escape($key) . '=' . uri_escape($data->{$key});
        }
        $url .= '?' . join('&', @params) if @params;
    }

    my $req = HTTP::Request->new($method => $url);

    # Add JSON body for POST/PUT
    if ($method =~ /^(POST|PUT|PATCH)$/ && $data) {
        $req->header('Content-Type' => 'application/json');
        $req->content(encode_json($data));
    }

    my $response = $self->ua->request($req);

    if ($response->is_success) {
        my $content = $response->decoded_content;
        return $content ? decode_json($content) : {};
    } else {
        my $error = $response->status_line;
        if ($response->content) {
            eval {
                my $json = decode_json($response->content);
                $error = $json->{error} || $json->{message} || $error;
            };
        }
        croak "API request failed: $error";
    }
}

1;

__END__

=head1 NAME

WebService::Example - Perl client for Example.com API

=head1 SYNOPSIS

    use WebService::Example;

    my $api = WebService::Example->new(
        api_key => 'your-api-key',
    );

    # Get a user
    my $user = $api->get_user(123);

    # Search users
    my $results = $api->search_users({
        name => 'Alice',
        status => 'active',
    });

    # Create a user
    my $new_user = $api->create_user({
        name => 'Bob',
        email => 'bob@example.com',
    });

=head1 DESCRIPTION

This module provides a Perl interface to the Example.com REST API.

=head1 METHODS

[Full documentation here...]

=head1 AUTHOR

Your Name <your.email@example.com>

=head1 COPYRIGHT AND LICENSE

This software is copyright (c) 2024 by Your Name.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.

=cut
```

## CPAN Module Evaluation Checklist

When choosing a CPAN module:

1. **Check last update date** - Recently maintained?
2. **Review test results** - Pass on your platform?
3. **Examine dependencies** - Reasonable dependency tree?
4. **Read bug reports** - Active issues?
5. **Look at documentation** - Well documented?
6. **Check ratings/reviews** - What do others say?
7. **Review source code** - Clean and understandable?
8. **Test coverage** - Good test suite?
9. **License compatibility** - Fits your needs?
10. **Author reputation** - Other quality modules?

## Conclusion

CPAN is more than a module repository—it's the collective knowledge and effort of thousands of Perl programmers over decades. By understanding how to navigate, evaluate, use, and contribute to CPAN, you join a community that values sharing, testing, and continuous improvement.

Remember: before writing new code, check CPAN. Someone may have already solved your problem, tested it across dozens of platforms, and documented it thoroughly. And when you solve a new problem, consider sharing it back. That's how CPAN grew from a handful of modules to the treasure trove it is today.

---

*Next: Object-Oriented Perl. We'll explore Perl's flexible approach to OOP, from basic blessed references to modern frameworks like Moo and Moose.*