# Chapter 18: Building Command-Line Tools

> "A good command-line tool is like a trusty wrench - it does one thing well, plays nicely with others, and doesn't require a manual to use." - Unix Philosophy, Perl Edition

Command-line tools are Perl's bread and butter. From simple scripts to complex applications, Perl excels at creating tools that system administrators and developers rely on daily. This chapter shows you how to build professional CLI tools with proper argument parsing, help text, configuration, and all the features users expect from modern command-line applications.

## Command-Line Argument Parsing

### Basic with Getopt::Long

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Getopt::Long;
use Pod::Usage;

# Define options
my %options = (
    verbose => 0,
    output  => '-',
    format  => 'json',
    limit   => 100,
);

# Parse command line
GetOptions(
    'verbose|v+'    => \$options{verbose},      # -v, -vv, -vvv
    'quiet|q'       => sub { $options{verbose} = -1 },
    'output|o=s'    => \$options{output},       # Requires string
    'format|f=s'    => \$options{format},       # --format json
    'limit|l=i'     => \$options{limit},        # Requires integer
    'dry-run|n'     => \$options{dry_run},      # Boolean flag
    'help|h|?'      => sub { pod2usage(1) },
    'man'           => sub { pod2usage(-verbose => 2) },
    'version'       => sub { say "Version 1.0"; exit },
) or pod2usage(2);

# Validate options
pod2usage("--format must be json, xml, or csv")
    unless $options{format} =~ /^(json|xml|csv)$/;

pod2usage("--limit must be positive")
    unless $options{limit} > 0;

# Process remaining arguments
my @files = @ARGV;
@files = ('-') unless @files;  # Default to STDIN

# Main logic
for my $file (@files) {
    process_file($file, \%options);
}

sub process_file {
    my ($file, $opts) = @_;

    say "Processing $file..." if $opts->{verbose} > 0;

    # Implementation...
}

__END__

=head1 NAME

mytool - Process files with various formats

=head1 SYNOPSIS

mytool [options] [file ...]

 Options:
   -v, --verbose     Increase verbosity (can be repeated)
   -q, --quiet       Suppress all output
   -o, --output FILE Write output to FILE (default: STDOUT)
   -f, --format FMT  Output format: json, xml, csv (default: json)
   -l, --limit N     Process at most N records (default: 100)
   -n, --dry-run     Don't actually process, just show what would be done
   -h, --help        Brief help message
   --man             Full documentation
   --version         Show version

=head1 DESCRIPTION

This tool processes input files and produces formatted output.

=cut
```

### Advanced Argument Parsing with Getopt::Long::Descriptive

```perl
use Getopt::Long::Descriptive;

my ($opt, $usage) = describe_options(
    'mytool %o <file>',
    [ 'verbose|v+', "increase verbosity" ],
    [ 'output|o=s', "output file", { default => '-' } ],
    [ 'format|f=s', "output format", {
        default => 'json',
        callbacks => {
            'valid format' => sub {
                $_[0] =~ /^(json|xml|csv)$/
            },
        },
    }],
    [],  # Blank line in help
    [ 'mode' => hidden => {
        one_of => [
            [ 'add|a'    => "add items" ],
            [ 'remove|r' => "remove items" ],
            [ 'list|l'   => "list items" ],
        ],
    }],
    [],
    [ 'help|h', "print usage message and exit", { shortcircuit => 1 } ],
);

print($usage->text), exit if $opt->help;

# Use options
say "Verbose level: " . $opt->verbose if $opt->verbose;
process_file($opt->output, $opt->format);
```

## Interactive Command-Line Tools

### Building Interactive Prompts

```perl
use Term::ANSIColor;
use Term::ReadLine;
use Term::Choose;

# Colored output
sub info {
    say colored(['bright_blue'], "[INFO] $_[0]");
}

sub warning {
    say colored(['yellow'], "[WARN] $_[0]");
}

sub error {
    say colored(['red'], "[ERROR] $_[0]");
}

sub success {
    say colored(['green'], "[✓] $_[0]");
}

# Interactive prompt
sub prompt {
    my ($question, $default) = @_;

    my $term = Term::ReadLine->new('MyApp');
    my $prompt = $question;
    $prompt .= " [$default]" if defined $default;
    $prompt .= ": ";

    my $answer = $term->readline($prompt);
    $answer = $default if !defined $answer || $answer eq '';

    return $answer;
}

# Yes/No confirmation
sub confirm {
    my ($question, $default) = @_;
    $default //= 'n';

    my $answer = prompt("$question (y/n)", $default);
    return $answer =~ /^y/i;
}

# Menu selection
sub menu {
    my ($title, @options) = @_;

    my $tc = Term::Choose->new({
        prompt => $title,
        layout => 2,
        mouse  => 1,
    });

    return $tc->choose([@options]);
}

# Password input
use Term::ReadKey;

sub get_password {
    my ($prompt) = @_;
    $prompt //= "Password";

    print "$prompt: ";
    ReadMode('noecho');  # Don't echo keystrokes

    my $password = ReadLine(0);
    ReadMode('restore');
    print "\n";

    chomp $password;
    return $password;
}

# Progress bar
use Term::ProgressBar;

sub show_progress {
    my ($total) = @_;

    my $progress = Term::ProgressBar->new({
        name  => 'Processing',
        count => $total,
        ETA   => 'linear',
    });

    for my $i (1..$total) {
        # Do work
        sleep(0.01);

        $progress->update($i);
    }

    $progress->finish;
}

# Usage example
info("Starting application");

my $name = prompt("Enter your name", $ENV{USER});
success("Welcome, $name!");

if (confirm("Do you want to continue?", 'y')) {
    my $choice = menu("Select an option:", qw(Add Remove List Quit));

    if ($choice && $choice ne 'Quit') {
        info("You selected: $choice");

        my $password = get_password("Enter password");
        show_progress(100);
    }
}

warning("Exiting application");
```

## Configuration Management

### Supporting Configuration Files

```perl
package App::Config;
use Config::Any;
use File::HomeDir;
use File::Spec;
use Carp;

sub new {
    my ($class, %args) = @_;

    my $self = bless {
        app_name => $args{app_name} || 'myapp',
        config => {},
    }, $class;

    $self->load_config();
    return $self;
}

sub load_config {
    my $self = shift;

    # Configuration file search paths
    my @config_paths = (
        File::Spec->catfile(File::HomeDir->my_home, "." . $self->{app_name}),
        File::Spec->catfile(File::HomeDir->my_home, "." . $self->{app_name}, "config"),
        File::Spec->catfile("/etc", $self->{app_name}, "config"),
        "./" . $self->{app_name} . ".conf",
    );

    # Load first config found
    for my $base_path (@config_paths) {
        my @files = map { "$base_path.$_" } qw(yaml yml json ini conf);

        for my $file (@files) {
            next unless -r $file;

            my $cfg = Config::Any->load_files({
                files => [$file],
                use_ext => 1,
            });

            if ($cfg && @$cfg) {
                $self->{config} = $cfg->[0]{$file};
                $self->{config_file} = $file;
                last;
            }
        }

        last if $self->{config_file};
    }

    # Merge with environment variables
    $self->merge_env_config();

    return $self->{config};
}

sub merge_env_config {
    my $self = shift;

    my $prefix = uc($self->{app_name}) . "_";

    for my $key (keys %ENV) {
        next unless $key =~ /^$prefix(.+)$/;

        my $config_key = lc($1);
        $config_key =~ s/_/./g;  # MYAPP_DATABASE_HOST -> database.host

        $self->set($config_key, $ENV{$key});
    }
}

sub get {
    my ($self, $key, $default) = @_;

    my @parts = split /\./, $key;
    my $value = $self->{config};

    for my $part (@parts) {
        return $default unless ref $value eq 'HASH';
        $value = $value->{$part};
        return $default unless defined $value;
    }

    return $value;
}

sub set {
    my ($self, $key, $value) = @_;

    my @parts = split /\./, $key;
    my $last = pop @parts;
    my $ref = $self->{config};

    for my $part (@parts) {
        $ref->{$part} //= {};
        $ref = $ref->{$part};
    }

    $ref->{$last} = $value;
}

# Usage
package main;

my $config = App::Config->new(app_name => 'mytool');

my $db_host = $config->get('database.host', 'localhost');
my $db_port = $config->get('database.port', 5432);
```

## Professional CLI Application Structure

### Complete CLI Application

```perl
#!/usr/bin/env perl
package MyApp::CLI;
use Modern::Perl '2023';
use Moo;
use Types::Standard qw(Str Int Bool HashRef);
use Getopt::Long::Descriptive;
use Term::ANSIColor;
use Try::Tiny;
use Log::Any '$log';
use Log::Any::Adapter;

# Attributes
has 'config' => (is => 'ro', isa => HashRef, default => sub { {} });
has 'verbose' => (is => 'rw', isa => Int, default => 0);
has 'dry_run' => (is => 'rw', isa => Bool, default => 0);
has 'output' => (is => 'rw', isa => Str, default => '-');

# Main entry point
sub run {
    my ($self, @args) = @_;

    local @ARGV = @args if @args;

    try {
        $self->parse_options();
        $self->setup_logging();
        $self->validate_environment();
        $self->execute();
    } catch {
        $self->error("Fatal error: $_");
        exit 1;
    };
}

sub parse_options {
    my $self = shift;

    my ($opt, $usage) = describe_options(
        '%c %o <command> [<args>]',
        [ 'verbose|v+', "increase verbosity" ],
        [ 'quiet|q', "suppress output" ],
        [ 'dry-run|n', "don't make changes" ],
        [ 'config|c=s', "config file" ],
        [ 'output|o=s', "output file", { default => '-' } ],
        [],
        [ 'help|h', "show help", { shortcircuit => 1 } ],
        [ 'version', "show version", { shortcircuit => 1 } ],
    );

    if ($opt->help) {
        print $usage->text;
        exit 0;
    }

    if ($opt->version) {
        say "MyApp version 1.0.0";
        exit 0;
    }

    # Store options
    $self->verbose($opt->verbose - ($opt->quiet ? 1 : 0));
    $self->dry_run($opt->dry_run);
    $self->output($opt->output);

    # Load config if specified
    if ($opt->config) {
        $self->load_config($opt->config);
    }

    # Get command
    $self->{command} = shift @ARGV || 'help';
    $self->{args} = [@ARGV];
}

sub setup_logging {
    my $self = shift;

    my $level = $self->verbose > 1 ? 'debug' :
                $self->verbose > 0 ? 'info' :
                $self->verbose < 0 ? 'error' : 'warning';

    Log::Any::Adapter->set('Stderr', log_level => $level);
}

sub execute {
    my $self = shift;

    my $command = $self->{command};
    my $method = "cmd_$command";

    if ($self->can($method)) {
        $self->$method(@{$self->{args}});
    } else {
        $self->error("Unknown command: $command");
        $self->cmd_help();
        exit 1;
    }
}

# Commands
sub cmd_help {
    my $self = shift;

    say <<'HELP';
Usage: myapp [options] <command> [<args>]

Commands:
    list      List all items
    add       Add a new item
    remove    Remove an item
    status    Show status
    help      Show this help

Options:
    -v, --verbose    Increase verbosity
    -q, --quiet      Suppress output
    -n, --dry-run    Don't make changes
    -c, --config     Config file
    -o, --output     Output file

Examples:
    myapp list
    myapp add --name "New Item"
    myapp remove item-123
    myapp -vv status
HELP
}

sub cmd_list {
    my $self = shift;

    $self->info("Listing items...");

    my @items = $self->get_items();

    if ($self->output eq '-') {
        for my $item (@items) {
            $self->print_item($item);
        }
    } else {
        $self->write_output(\@items);
    }

    $self->success("Listed " . scalar(@items) . " items");
}

sub cmd_add {
    my ($self, @args) = @_;

    my ($opt, $usage) = describe_options(
        'add %o',
        [ 'name|n=s', "item name", { required => 1 } ],
        [ 'description|d=s', "item description" ],
        [ 'tags|t=s@', "tags" ],
    );

    my $item = {
        name => $opt->name,
        description => $opt->description || '',
        tags => $opt->tags || [],
        created => time(),
    };

    if ($self->dry_run) {
        $self->info("Would add item: " . $item->{name});
    } else {
        $self->add_item($item);
        $self->success("Added item: " . $item->{name});
    }
}

sub cmd_remove {
    my ($self, $id) = @_;

    unless ($id) {
        $self->error("Item ID required");
        exit 1;
    }

    if ($self->dry_run) {
        $self->info("Would remove item: $id");
    } else {
        $self->remove_item($id);
        $self->success("Removed item: $id");
    }
}

sub cmd_status {
    my $self = shift;

    my $status = $self->get_status();

    say colored(['bold'], "System Status");
    say "=" x 40;

    for my $key (sort keys %$status) {
        my $value = $status->{$key};
        my $color = $value eq 'OK' ? 'green' :
                   $value eq 'WARNING' ? 'yellow' : 'red';

        printf "%-20s %s\n", $key, colored([$color], $value);
    }
}

# Utility methods
sub info {
    my ($self, $msg) = @_;
    return if $self->verbose < 0;
    say colored(['cyan'], "[INFO] $msg");
}

sub warning {
    my ($self, $msg) = @_;
    say STDERR colored(['yellow'], "[WARN] $msg");
}

sub error {
    my ($self, $msg) = @_;
    say STDERR colored(['red'], "[ERROR] $msg");
}

sub success {
    my ($self, $msg) = @_;
    return if $self->verbose < 0;
    say colored(['green'], "[✓] $msg");
}

sub debug {
    my ($self, $msg) = @_;
    return unless $self->verbose > 1;
    say colored(['gray'], "[DEBUG] $msg");
}

# Stub methods for demonstration
sub get_items { return map { { id => $_, name => "Item $_" } } 1..5 }
sub add_item { }
sub remove_item { }
sub get_status { return { Database => 'OK', Cache => 'OK', Queue => 'WARNING' } }
sub validate_environment { }
sub load_config { }
sub print_item { say "  - $_[1]{name}" }
sub write_output { }

# Script entry point
package main;

unless (caller) {
    my $app = MyApp::CLI->new();
    $app->run(@ARGV);
}

1;
```

## Creating Distributable Tools

### Using App::FatPacker

```perl
# Create a single-file executable
# Install: cpanm App::FatPacker

# Step 1: Trace dependencies
# fatpack trace script.pl

# Step 2: Pack dependencies
# fatpack packlists-for `cat fatpacker.trace` > packlists
# fatpack tree `cat packlists`

# Step 3: Create fatpacked script
# fatpack file script.pl > script_standalone.pl

# Or use this helper script:
#!/usr/bin/env perl
use strict;
use warnings;

my $script = shift or die "Usage: $0 <script.pl>\n";
my $output = $script;
$output =~ s/\.pl$/_standalone.pl/;

system("fatpack trace $script");
system("fatpack packlists-for `cat fatpacker.trace` > packlists");
system("fatpack tree `cat packlists`");
system("fatpack file $script > $output");

chmod 0755, $output;
unlink 'fatpacker.trace', 'packlists';
system("rm -rf fatlib");

print "Created standalone script: $output\n";
```

### Creating a CPAN Distribution

```perl
# Use Module::Starter
# cpanm Module::Starter

# Create new distribution
# module-starter --module=App::MyTool \
#   --author="Your Name" \
#   --email=you@example.com \
#   --builder=Module::Build

# Directory structure:
# App-MyTool/
# ├── Build.PL
# ├── Changes
# ├── lib/
# │   └── App/
# │       └── MyTool.pm
# ├── script/
# │   └── mytool
# ├── t/
# │   ├── 00-load.t
# │   └── 01-basic.t
# ├── MANIFEST
# └── README

# Build.PL
use Module::Build;

my $builder = Module::Build->new(
    module_name         => 'App::MyTool',
    license             => 'perl',
    dist_author         => 'Your Name <you@example.com>',
    dist_version_from   => 'lib/App/MyTool.pm',
    script_files        => ['script/mytool'],
    requires => {
        'perl'          => '5.016',
        'Getopt::Long'  => 0,
        'Pod::Usage'    => 0,
    },
    test_requires => {
        'Test::More' => 0,
    },
    add_to_cleanup      => [ 'App-MyTool-*' ],
    meta_merge => {
        resources => {
            repository => 'https://github.com/you/App-MyTool',
        },
    },
);

$builder->create_build_script();
```

## Real-World CLI Tool Example

```perl
#!/usr/bin/env perl
# loganalyzer - Advanced log analysis tool
use Modern::Perl '2023';
use Getopt::Long::Descriptive;
use Time::Piece;
use JSON::XS;
use Text::Table;

# Parse command line
my ($opt, $usage) = describe_options(
    '%c %o <logfile> ...',
    [ 'pattern|p=s', "search pattern (regex)" ],
    [ 'from|f=s', "start date (YYYY-MM-DD)" ],
    [ 'to|t=s', "end date (YYYY-MM-DD)" ],
    [ 'level|l=s@', "log levels to include" ],
    [ 'exclude|x=s@', "patterns to exclude" ],
    [],
    [ 'output-format|o=s', "output format", {
        default => 'table',
        callbacks => {
            'valid format' => sub { $_[0] =~ /^(table|json|csv)$/ }
        }
    }],
    [ 'stats|s', "show statistics" ],
    [ 'follow|F', "follow file (like tail -f)" ],
    [],
    [ 'help|h', "show help", { shortcircuit => 1 } ],
);

print($usage->text), exit if $opt->help;
die "No log files specified\n" unless @ARGV;

# Main processor
my $analyzer = LogAnalyzer->new(
    pattern => $opt->pattern ? qr/$opt->{pattern}/ : undef,
    from_date => $opt->from ? parse_date($opt->from) : undef,
    to_date => $opt->to ? parse_date($opt->to) : undef,
    levels => $opt->level ? { map { $_ => 1 } @{$opt->level} } : undef,
    exclude => $opt->exclude ? [map { qr/$_/ } @{$opt->exclude}] : [],
    output_format => $opt->output_format,
    show_stats => $opt->stats,
    follow => $opt->follow,
);

$analyzer->process_files(@ARGV);

package LogAnalyzer;
use Moo;
use File::Tail;

has [qw(pattern from_date to_date levels exclude)] => (is => 'ro');
has 'output_format' => (is => 'ro', default => 'table');
has 'show_stats' => (is => 'ro', default => 0);
has 'follow' => (is => 'ro', default => 0);
has 'stats' => (is => 'rw', default => sub { {} });
has 'results' => (is => 'rw', default => sub { [] });

sub process_files {
    my ($self, @files) = @_;

    if ($self->follow && @files == 1) {
        $self->follow_file($files[0]);
    } else {
        for my $file (@files) {
            $self->process_file($file);
        }
        $self->output_results();
    }
}

sub process_file {
    my ($self, $file) = @_;

    open my $fh, '<', $file or die "Can't open $file: $!";

    while (my $line = <$fh>) {
        chomp $line;
        $self->process_line($line);
    }

    close $fh;
}

sub follow_file {
    my ($self, $file) = @_;

    my $tail = File::Tail->new($file);

    while (defined(my $line = $tail->read)) {
        chomp $line;
        my $entry = $self->process_line($line);

        if ($entry) {
            $self->output_entry($entry);
        }
    }
}

sub process_line {
    my ($self, $line) = @_;

    # Skip if matches exclude pattern
    for my $exclude (@{$self->exclude}) {
        return if $line =~ $exclude;
    }

    # Parse log line (customize for your format)
    my $entry = $self->parse_log_line($line);
    return unless $entry;

    # Apply filters
    return if $self->pattern && $line !~ $self->pattern;
    return if $self->levels && !$self->levels->{$entry->{level}};
    return if $self->from_date && $entry->{timestamp} < $self->from_date;
    return if $self->to_date && $entry->{timestamp} > $self->to_date;

    # Update statistics
    $self->update_stats($entry) if $self->show_stats;

    # Store result
    push @{$self->results}, $entry unless $self->follow;

    return $entry;
}

sub parse_log_line {
    my ($self, $line) = @_;

    # Example: 2024-01-15 10:30:45 [ERROR] Connection timeout
    if ($line =~ /^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+\[(\w+)\]\s+(.+)$/) {
        return {
            timestamp => $1,
            level => $2,
            message => $3,
            raw => $line,
        };
    }

    return undef;
}

sub update_stats {
    my ($self, $entry) = @_;

    $self->stats->{total}++;
    $self->stats->{by_level}{$entry->{level}}++;
}

sub output_results {
    my $self = shift;

    if ($self->output_format eq 'json') {
        say encode_json($self->results);
    } elsif ($self->output_format eq 'csv') {
        $self->output_csv();
    } else {
        $self->output_table();
    }

    $self->output_statistics() if $self->show_stats;
}

sub output_entry {
    my ($self, $entry) = @_;

    if ($self->output_format eq 'json') {
        say encode_json($entry);
    } else {
        say "$entry->{timestamp} [$entry->{level}] $entry->{message}";
    }
}

sub output_table {
    my $self = shift;

    my $table = Text::Table->new(
        "Timestamp", "Level", "Message"
    );

    for my $entry (@{$self->results}) {
        $table->load([$entry->{timestamp}, $entry->{level}, $entry->{message}]);
    }

    print $table;
}

sub output_csv {
    my $self = shift;

    say "Timestamp,Level,Message";
    for my $entry (@{$self->results}) {
        say qq("$entry->{timestamp}","$entry->{level}","$entry->{message}");
    }
}

sub output_statistics {
    my $self = shift;

    say "\nStatistics:";
    say "-" x 40;
    say "Total entries: " . ($self->stats->{total} // 0);

    if ($self->stats->{by_level}) {
        say "\nBy Level:";
        for my $level (sort keys %{$self->stats->{by_level}}) {
            printf "  %-10s: %d\n", $level, $self->stats->{by_level}{$level};
        }
    }
}

sub parse_date {
    my $date = shift;
    return Time::Piece->strptime($date, '%Y-%m-%d');
}
```

## Best Practices

1. **Follow Unix philosophy** - Do one thing well
2. **Support standard conventions** - Use -, --, read from STDIN
3. **Provide helpful error messages** - Guide users to success
4. **Include examples in help** - Show, don't just tell
5. **Support configuration files** - For complex tools
6. **Make output parseable** - Support JSON/CSV for scripting
7. **Use exit codes properly** - 0 for success, non-zero for errors
8. **Support verbose and quiet modes** - Let users control output
9. **Handle signals gracefully** - Clean up on SIGINT/SIGTERM
10. **Test your CLI** - Use Test::Cmd or similar

## Conclusion

Building command-line tools in Perl is a joy. The language's text processing power, combined with excellent CPAN modules for argument parsing and terminal interaction, makes it ideal for creating the kind of tools system administrators and developers use every day.

Remember: a good CLI tool feels intuitive to use, provides helpful feedback, and plays well with other tools in the Unix tradition. Perl gives you all the pieces—it's up to you to assemble them thoughtfully.

---

*Next: System monitoring and alerting scripts. We'll build tools that keep watch over your infrastructure and alert you when things go wrong.*