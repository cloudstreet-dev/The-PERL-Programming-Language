# Chapter 10: Process Management and System Commands

> "Perl is the glue that holds the Internet together - and the system administrator's Swiss Army knife." - Larry Wall

System administration isn't just about files and text—it's about controlling processes, executing commands, and orchestrating system resources. Perl excels at being the conductor of this symphony, providing fine-grained control over process creation, inter-process communication, and system interaction. This chapter shows you how to wield these powers responsibly.

## Executing System Commands

### The Many Ways to Run Commands

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Method 1: system() - Returns exit status
my $status = system('ls', '-la', '/tmp');
if ($status == 0) {
    say "Command succeeded";
} else {
    say "Command failed with status: " . ($status >> 8);
}

# Method 2: Backticks - Captures output
my $output = `ls -la /tmp`;
say "Output: $output";

# Method 3: qx// - Same as backticks but cleaner
my $users = qx/who/;
say "Currently logged in:\n$users";

# Method 4: open() with pipe - More control
open my $pipe, '-|', 'ps', 'aux' or die "Can't run ps: $!";
while (<$pipe>) {
    print if /perl/i;  # Only show perl processes
}
close $pipe;

# Method 5: IPC::Run for complex scenarios
use IPC::Run qw(run timeout);

my ($in, $out, $err);
run ['grep', 'pattern', 'file.txt'], \$in, \$out, \$err, timeout(10)
    or die "grep failed: $?";
say "Matches: $out";
say "Errors: $err" if $err;
```

### Safe Command Execution

```perl
# NEVER do this - shell injection vulnerability!
my $user_input = shift @ARGV;
system("ls $user_input");  # DANGEROUS!

# DO this instead - bypass shell
system('ls', $user_input);  # Safe - no shell interpretation

# Or validate input
if ($user_input =~ /^[\w\-\.\/]+$/) {
    system("ls $user_input");
} else {
    die "Invalid input";
}

# Safe command builder
sub safe_command {
    my ($cmd, @args) = @_;

    # Validate command
    my %allowed = map { $_ => 1 } qw(ls ps df du top);
    die "Command not allowed: $cmd" unless $allowed{$cmd};

    # Escape arguments
    @args = map { quotemeta($_) } @args;

    return system($cmd, @args);
}

# Taint mode for extra safety
#!/usr/bin/perl -T
use strict;
use warnings;

# With taint mode, Perl won't let you use tainted data in dangerous operations
my $user_input = $ENV{USER_DATA};  # Tainted
system("echo $user_input");  # Dies with taint error

# Must explicitly untaint
if ($user_input =~ /^([\w\s]+)$/) {
    my $clean = $1;  # Untainted
    system("echo $clean");  # Now safe
}
```

## Process Management

### Fork and Exec

```perl
# Basic fork
my $pid = fork();
die "Fork failed: $!" unless defined $pid;

if ($pid == 0) {
    # Child process
    say "I'm the child with PID $$";
    exec('sleep', '10') or die "Exec failed: $!";
} else {
    # Parent process
    say "I'm the parent, child PID is $pid";
    my $kid = waitpid($pid, 0);
    say "Child $kid exited with status " . ($? >> 8);
}

# Fork multiple children
sub fork_children {
    my ($count, $work) = @_;
    my @pids;

    for (1..$count) {
        my $pid = fork();
        die "Fork failed: $!" unless defined $pid;

        if ($pid == 0) {
            # Child
            $work->($_);
            exit(0);
        } else {
            # Parent
            push @pids, $pid;
        }
    }

    # Wait for all children
    for my $pid (@pids) {
        waitpid($pid, 0);
    }
}

# Use it
fork_children(5, sub {
    my $num = shift;
    say "Worker $num (PID $$) starting";
    sleep(rand(5));
    say "Worker $num done";
});
```

### Process Monitoring

```perl
# Monitor system processes
sub get_process_info {
    my ($pid) = @_;

    return unless kill(0, $pid);  # Check if process exists

    # Read from /proc (Linux)
    if (-d "/proc/$pid") {
        my $info = {};

        # Command line
        if (open my $fh, '<', "/proc/$pid/cmdline") {
            local $/;
            $info->{cmdline} = <$fh>;
            $info->{cmdline} =~ s/\0/ /g;
            close $fh;
        }

        # Status
        if (open my $fh, '<', "/proc/$pid/stat") {
            my $stat = <$fh>;
            my @fields = split /\s+/, $stat;
            $info->{state} = $fields[2];
            $info->{ppid} = $fields[3];
            $info->{utime} = $fields[13];
            $info->{stime} = $fields[14];
            close $fh;
        }

        # Memory
        if (open my $fh, '<', "/proc/$pid/status") {
            while (<$fh>) {
                if (/^VmRSS:\s+(\d+)/) {
                    $info->{memory} = $1 * 1024;  # Convert to bytes
                }
            }
            close $fh;
        }

        return $info;
    }

    # Fallback to ps command
    my $ps = `ps -p $pid -o pid,ppid,user,command`;
    return parse_ps_output($ps);
}

# Process tree
sub show_process_tree {
    my ($pid, $indent) = @_;
    $pid //= $$;
    $indent //= 0;

    my $info = get_process_info($pid);
    return unless $info;

    print " " x $indent;
    say "PID $pid: $info->{cmdline}";

    # Find children
    my @children = find_children($pid);
    for my $child (@children) {
        show_process_tree($child, $indent + 2);
    }
}

sub find_children {
    my ($ppid) = @_;
    my @children;

    opendir my $dh, '/proc' or return;
    while (my $dir = readdir $dh) {
        next unless $dir =~ /^\d+$/;

        if (open my $fh, '<', "/proc/$dir/stat") {
            my $stat = <$fh>;
            my @fields = split /\s+/, $stat;
            push @children, $dir if $fields[3] == $ppid;
            close $fh;
        }
    }
    closedir $dh;

    return @children;
}
```

## Inter-Process Communication (IPC)

### Pipes

```perl
# Simple pipe
pipe(my $reader, my $writer) or die "Pipe failed: $!";

my $pid = fork();
die "Fork failed: $!" unless defined $pid;

if ($pid == 0) {
    # Child writes
    close $reader;
    print $writer "Hello from child\n";
    print $writer "PID: $$\n";
    close $writer;
    exit(0);
} else {
    # Parent reads
    close $writer;
    while (<$reader>) {
        print "Parent received: $_";
    }
    close $reader;
    waitpid($pid, 0);
}

# Bidirectional communication with IPC::Open2
use IPC::Open2;

my ($reader, $writer);
my $pid = open2($reader, $writer, 'bc', '-l')  # Calculator
    or die "Can't run bc: $!";

# Send calculations
print $writer "2 + 2\n";
print $writer "scale=2; 22/7\n";
print $writer "quit\n";
close $writer;

# Read results
while (<$reader>) {
    chomp;
    say "Result: $_";
}
close $reader;
waitpid($pid, 0);
```

### Shared Memory and Semaphores

```perl
use IPC::SysV qw(IPC_PRIVATE IPC_CREAT IPC_RMID S_IRUSR S_IWUSR);
use IPC::SharedMem;
use IPC::Semaphore;

# Create shared memory
my $shm = IPC::SharedMem->new(
    IPC_PRIVATE,
    1024,
    S_IRUSR | S_IWUSR | IPC_CREAT
) or die "Can't create shared memory: $!";

# Create semaphore for synchronization
my $sem = IPC::Semaphore->new(
    IPC_PRIVATE,
    1,
    S_IRUSR | S_IWUSR | IPC_CREAT
) or die "Can't create semaphore: $!";

$sem->setval(0, 1);  # Initialize semaphore

# Fork a child
my $pid = fork();
die "Fork failed: $!" unless defined $pid;

if ($pid == 0) {
    # Child process
    for (1..5) {
        $sem->op(0, -1, 0);  # Wait (P operation)

        my $data = $shm->read(0, 1024);
        $data =~ s/\0+$//;  # Remove null padding
        say "Child read: $data";

        $shm->write("Child message $_", 0, 1024);

        $sem->op(0, 1, 0);   # Signal (V operation)
        sleep(1);
    }
    exit(0);
} else {
    # Parent process
    for (1..5) {
        $sem->op(0, -1, 0);  # Wait

        $shm->write("Parent message $_", 0, 1024);

        $sem->op(0, 1, 0);   # Signal
        sleep(1);

        $sem->op(0, -1, 0);  # Wait

        my $data = $shm->read(0, 1024);
        $data =~ s/\0+$//;
        say "Parent read: $data";

        $sem->op(0, 1, 0);   # Signal
    }

    waitpid($pid, 0);

    # Cleanup
    $shm->remove;
    $sem->remove;
}
```

## Signal Handling

### Basic Signal Handling

```perl
# Set up signal handlers
$SIG{INT} = sub {
    say "\nReceived SIGINT, cleaning up...";
    cleanup();
    exit(0);
};

$SIG{TERM} = sub {
    say "Received SIGTERM, shutting down gracefully...";
    cleanup();
    exit(0);
};

$SIG{HUP} = sub {
    say "Received SIGHUP, reloading configuration...";
    reload_config();
};

$SIG{CHLD} = 'IGNORE';  # Auto-reap children

# Or use a dispatch table
my %sig_handlers = (
    INT  => \&handle_interrupt,
    TERM => \&handle_terminate,
    HUP  => \&handle_reload,
    USR1 => \&handle_user1,
    USR2 => \&handle_user2,
);

for my $sig (keys %sig_handlers) {
    $SIG{$sig} = $sig_handlers{$sig};
}

# Send signals to other processes
kill 'TERM', $pid;      # Send SIGTERM
kill 'USR1', @pids;     # Send to multiple processes
kill 0, $pid;           # Check if process exists

# Alarm signal for timeouts
$SIG{ALRM} = sub { die "Timeout!" };
alarm(10);  # 10 second timeout

eval {
    # Long running operation
    lengthy_operation();
    alarm(0);  # Cancel alarm
};
if ($@ && $@ =~ /Timeout/) {
    warn "Operation timed out";
}
```

### Safe Signal Handling

```perl
# Safe signal handling with POSIX
use POSIX qw(:signal_h);

# Block signals during critical sections
my $sigset = POSIX::SigSet->new(SIGINT, SIGTERM);
sigprocmask(SIG_BLOCK, $sigset);

# Critical section
do_critical_work();

# Unblock signals
sigprocmask(SIG_UNBLOCK, $sigset);

# Or use local signal handlers
sub critical_operation {
    local $SIG{INT} = 'IGNORE';
    local $SIG{TERM} = 'IGNORE';

    # Critical work here
}
```

## System Information

### Gathering System Stats

```perl
# System information
use Sys::Hostname;
use POSIX qw(uname);

my $hostname = hostname();
my ($sysname, $nodename, $release, $version, $machine) = uname();

say "Hostname: $hostname";
say "System: $sysname $release on $machine";

# Load average
sub get_load_average {
    if (open my $fh, '<', '/proc/loadavg') {
        my $line = <$fh>;
        close $fh;
        my @loads = split /\s+/, $line;
        return @loads[0..2];
    }
    return;
}

# Memory information
sub get_memory_info {
    my %mem;

    if (open my $fh, '<', '/proc/meminfo') {
        while (<$fh>) {
            if (/^(\w+):\s+(\d+)/) {
                $mem{$1} = $2 * 1024;  # Convert to bytes
            }
        }
        close $fh;
    }

    return \%mem;
}

# Disk usage
sub get_disk_usage {
    my @disks;

    open my $pipe, '-|', 'df', '-k' or die "Can't run df: $!";
    my $header = <$pipe>;  # Skip header

    while (<$pipe>) {
        my @fields = split /\s+/;
        next unless @fields >= 6;

        push @disks, {
            filesystem => $fields[0],
            size => $fields[1] * 1024,
            used => $fields[2] * 1024,
            available => $fields[3] * 1024,
            percent => $fields[4],
            mount => $fields[5],
        };
    }
    close $pipe;

    return \@disks;
}

# CPU information
sub get_cpu_info {
    my @cpus;

    if (open my $fh, '<', '/proc/cpuinfo') {
        my $cpu = {};
        while (<$fh>) {
            chomp;
            if (/^processor\s*:\s*(\d+)/) {
                push @cpus, $cpu if %$cpu;
                $cpu = { id => $1 };
            } elsif (/^([^:]+)\s*:\s*(.+)/) {
                my ($key, $value) = ($1, $2);
                $key =~ s/\s+/_/g;
                $cpu->{lc($key)} = $value;
            }
        }
        push @cpus, $cpu if %$cpu;
        close $fh;
    }

    return \@cpus;
}
```

## Daemon Processes

### Creating a Daemon

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use POSIX qw(setsid);

sub daemonize {
    my ($pidfile) = @_;

    # Fork and exit parent
    my $pid = fork();
    exit(0) if $pid;
    die "Fork failed: $!" unless defined $pid;

    # Create new session
    setsid() or die "Can't start new session: $!";

    # Fork again to ensure we can't acquire a controlling terminal
    $pid = fork();
    exit(0) if $pid;
    die "Fork failed: $!" unless defined $pid;

    # Change working directory
    chdir('/') or die "Can't chdir to /: $!";

    # Clear file creation mask
    umask(0);

    # Close file descriptors
    close(STDIN);
    close(STDOUT);
    close(STDERR);

    # Reopen standard file descriptors
    open(STDIN, '<', '/dev/null') or die "Can't read /dev/null: $!";
    open(STDOUT, '>', '/dev/null') or die "Can't write /dev/null: $!";
    open(STDERR, '>', '/dev/null') or die "Can't write /dev/null: $!";

    # Write PID file
    if ($pidfile) {
        open my $fh, '>', $pidfile or die "Can't write PID file: $!";
        print $fh $$;
        close $fh;
    }

    return $$;
}

# Daemon with logging
sub daemon_with_logging {
    my ($name, $logfile) = @_;

    # Daemonize
    my $pidfile = "/var/run/$name.pid";
    daemonize($pidfile);

    # Open log file
    open my $log, '>>', $logfile or die "Can't open log: $!";
    $log->autoflush(1);

    # Redirect STDOUT and STDERR to log
    open(STDOUT, '>&', $log) or die "Can't dup STDOUT: $!";
    open(STDERR, '>&', $log) or die "Can't dup STDERR: $!";

    # Set up signal handlers
    $SIG{TERM} = sub {
        say "Daemon shutting down";
        unlink $pidfile;
        exit(0);
    };

    $SIG{HUP} = sub {
        say "Reloading configuration";
        # Reload config here
    };

    # Main daemon loop
    say "Daemon started with PID $$";
    while (1) {
        # Do daemon work
        do_work();
        sleep(60);
    }
}

sub do_work {
    my $timestamp = localtime();
    say "[$timestamp] Performing scheduled work";
    # Your daemon logic here
}
```

## Real-World Example: Process Manager

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package ProcessManager;
use Time::HiRes qw(sleep);
use POSIX qw(:sys_wait_h);

sub new($class, $config) {
    return bless {
        config => $config,
        workers => {},
        running => 0,
    }, $class;
}

sub start($self) {
    $self->setup_signals();
    $self->spawn_workers();
    $self->monitor_loop();
}

sub setup_signals($self) {
    $SIG{TERM} = sub { $self->shutdown() };
    $SIG{INT} = sub { $self->shutdown() };
    $SIG{HUP} = sub { $self->reload() };
    $SIG{CHLD} = sub { $self->reap_children() };
}

sub spawn_workers($self) {
    my $count = $self->{config}{worker_count} // 5;

    for (1..$count) {
        $self->spawn_worker();
    }
}

sub spawn_worker($self) {
    my $pid = fork();
    die "Fork failed: $!" unless defined $pid;

    if ($pid == 0) {
        # Child process
        $0 = "$0 [worker]";  # Change process name
        $self->worker_loop();
        exit(0);
    } else {
        # Parent process
        $self->{workers}{$pid} = {
            pid => $pid,
            started => time(),
            status => 'running',
        };
        $self->{running}++;
        say "Spawned worker $pid";
    }
}

sub worker_loop($self) {
    # Reset signal handlers in worker
    $SIG{TERM} = sub { exit(0) };
    $SIG{INT} = 'DEFAULT';
    $SIG{HUP} = 'DEFAULT';

    while (1) {
        # Simulate work
        my $task = $self->get_task();
        if ($task) {
            $self->process_task($task);
        } else {
            sleep(1);
        }
    }
}

sub get_task($self) {
    # In real implementation, get from queue, database, etc.
    return rand() > 0.5 ? { id => int(rand(1000)), type => 'process' } : undef;
}

sub process_task($self, $task) {
    say "[$$] Processing task $task->{id}";
    sleep(rand(3));  # Simulate work
    say "[$$] Completed task $task->{id}";
}

sub monitor_loop($self) {
    while (1) {
        $self->check_workers();
        $self->report_status();
        sleep(5);
    }
}

sub check_workers($self) {
    for my $pid (keys %{$self->{workers}}) {
        unless (kill 0, $pid) {
            # Worker died unexpectedly
            say "Worker $pid died, respawning";
            delete $self->{workers}{$pid};
            $self->{running}--;
            $self->spawn_worker();
        }
    }
}

sub reap_children($self) {
    while ((my $pid = waitpid(-1, WNOHANG)) > 0) {
        my $status = $? >> 8;
        say "Reaped child $pid with status $status";

        if (exists $self->{workers}{$pid}) {
            delete $self->{workers}{$pid};
            $self->{running}--;

            # Respawn if not shutting down
            unless ($self->{shutting_down}) {
                $self->spawn_worker();
            }
        }
    }
}

sub report_status($self) {
    my $now = time();
    say "\n=== Status Report ===";
    say "Workers: $self->{running}";

    for my $pid (sort { $a <=> $b } keys %{$self->{workers}}) {
        my $worker = $self->{workers}{$pid};
        my $uptime = $now - $worker->{started};
        say "  PID $pid: running for ${uptime}s";
    }
}

sub reload($self) {
    say "Reloading configuration";
    # Re-read config file
    # Adjust worker count if needed
}

sub shutdown($self) {
    say "\nShutting down...";
    $self->{shutting_down} = 1;

    # Send TERM to all workers
    for my $pid (keys %{$self->{workers}}) {
        say "Terminating worker $pid";
        kill 'TERM', $pid;
    }

    # Wait for workers to exit
    my $timeout = 10;
    while ($self->{running} > 0 && $timeout-- > 0) {
        sleep(1);
    }

    # Force kill if necessary
    if ($self->{running} > 0) {
        for my $pid (keys %{$self->{workers}}) {
            say "Force killing worker $pid";
            kill 'KILL', $pid;
        }
    }

    say "Shutdown complete";
    exit(0);
}

package main;

# Run the process manager
my $config = {
    worker_count => 3,
    max_tasks_per_worker => 100,
    worker_timeout => 300,
};

my $manager = ProcessManager->new($config);
$manager->start();
```

## Best Practices

1. **Always check return values** - System calls can fail
2. **Use list form of system()** - Avoids shell injection
3. **Reap zombie processes** - Set up SIGCHLD handler
4. **Handle signals gracefully** - Clean shutdown is important
5. **Use absolute paths in daemons** - Working directory may change
6. **Log daemon activities** - You can't see stdout in a daemon
7. **Implement proper locking** - Prevent multiple instances
8. **Test signal handling** - Make sure cleanup works
9. **Monitor resource usage** - Prevent resource leaks
10. **Document process relationships** - Who spawns whom

## Conclusion

Process management and system interaction are where Perl proves its worth as a system administration language. Whether you're orchestrating complex workflows, building robust daemons, or simply automating system tasks, Perl provides the tools to do it efficiently and reliably.

Remember: with great power comes great responsibility. Always validate inputs, handle errors gracefully, and test thoroughly—especially when your code has system-level access.

---

*Next: Network programming and web scraping. We'll explore how Perl handles network protocols, builds clients and servers, and extracts data from the web.*