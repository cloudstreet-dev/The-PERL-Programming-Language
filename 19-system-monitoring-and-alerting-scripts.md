# Chapter 19: System Monitoring and Alerting Scripts

> "The best time to find a problem is before your users do. The second best time is right now." - SRE Wisdom

System monitoring isn't just about knowing when things break—it's about understanding trends, preventing problems, and maintaining service quality. Perl's combination of system access, text processing, and networking makes it perfect for building monitoring solutions that keep your infrastructure healthy. This chapter shows you how to build monitoring scripts that watch, measure, alert, and even self-heal.

## System Metrics Collection

### CPU and Memory Monitoring

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Time::HiRes qw(time sleep);

package SystemMonitor;

sub new {
    my $class = shift;
    return bless {
        samples => [],
        max_samples => 60,  # Keep 1 minute of data
    }, $class;
}

sub get_cpu_usage {
    my $self = shift;

    if ($^O eq 'linux') {
        return $self->_get_linux_cpu();
    } elsif ($^O eq 'darwin') {
        return $self->_get_macos_cpu();
    } else {
        die "Unsupported OS: $^O";
    }
}

sub _get_linux_cpu {
    my $self = shift;

    open my $fh, '<', '/proc/stat' or die "Can't read /proc/stat: $!";
    my $line = <$fh>;
    close $fh;

    if ($line =~ /^cpu\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s+(\d+)/) {
        my ($user, $nice, $system, $idle, $iowait) = ($1, $2, $3, $4, $5);
        my $total = $user + $nice + $system + $idle + $iowait;
        my $active = $user + $nice + $system;

        # Calculate percentage based on previous sample
        if ($self->{last_cpu}) {
            my $total_diff = $total - $self->{last_cpu}{total};
            my $active_diff = $active - $self->{last_cpu}{active};

            my $usage = $total_diff > 0 ?
                ($active_diff / $total_diff) * 100 : 0;

            $self->{last_cpu} = { total => $total, active => $active };
            return sprintf("%.2f", $usage);
        }

        $self->{last_cpu} = { total => $total, active => $active };
        return 0;
    }
}

sub _get_macos_cpu {
    my $self = shift;

    my $output = `top -l 1 -n 0`;
    if ($output =~ /CPU usage:\s*([\d.]+)%\s*user,\s*([\d.]+)%\s*sys/) {
        return sprintf("%.2f", $1 + $2);
    }
    return 0;
}

sub get_memory_info {
    my $self = shift;

    if ($^O eq 'linux') {
        return $self->_get_linux_memory();
    } elsif ($^O eq 'darwin') {
        return $self->_get_macos_memory();
    }
}

sub _get_linux_memory {
    my $self = shift;
    my %mem;

    open my $fh, '<', '/proc/meminfo' or die "Can't read /proc/meminfo: $!";
    while (<$fh>) {
        if (/^(\w+):\s+(\d+)\s+kB/) {
            $mem{$1} = $2 * 1024;  # Convert to bytes
        }
    }
    close $fh;

    return {
        total => $mem{MemTotal},
        free => $mem{MemFree},
        available => $mem{MemAvailable} // $mem{MemFree},
        used => $mem{MemTotal} - ($mem{MemAvailable} // $mem{MemFree}),
        cached => $mem{Cached},
        buffers => $mem{Buffers},
        swap_total => $mem{SwapTotal},
        swap_free => $mem{SwapFree},
    };
}

sub _get_macos_memory {
    my $self = shift;

    my $vm_stat = `vm_stat`;
    my %stats;

    while ($vm_stat =~ /([^:]+):\s*(\d+)/g) {
        $stats{$1} = $2;
    }

    my $page_size = 4096;  # Usually 4KB
    my $total = `sysctl -n hw.memsize`;
    chomp $total;

    return {
        total => $total,
        free => $stats{'Pages free'} * $page_size,
        used => $total - ($stats{'Pages free'} * $page_size),
        cached => $stats{'Pages cached'} * $page_size,
    };
}

sub get_disk_usage {
    my $self = shift;
    my @disks;

    my @df_output = `df -k 2>/dev/null`;
    shift @df_output;  # Skip header

    for my $line (@df_output) {
        my @fields = split /\s+/, $line;
        next unless @fields >= 6;
        next if $fields[0] =~ /^(tmpfs|devfs|overlay)/;

        push @disks, {
            filesystem => $fields[0],
            total => $fields[1] * 1024,
            used => $fields[2] * 1024,
            available => $fields[3] * 1024,
            percent => $fields[4],
            mount => $fields[5],
        };
    }

    return \@disks;
}

sub get_load_average {
    my $self = shift;

    if (open my $fh, '<', '/proc/loadavg') {
        my $line = <$fh>;
        close $fh;
        my @loads = split /\s+/, $line;
        return @loads[0..2];
    } else {
        # Fallback to uptime command
        my $uptime = `uptime`;
        if ($uptime =~ /load average:\s*([\d.]+),\s*([\d.]+),\s*([\d.]+)/) {
            return ($1, $2, $3);
        }
    }
    return (0, 0, 0);
}

sub get_network_stats {
    my $self = shift;
    my %stats;

    if ($^O eq 'linux') {
        open my $fh, '<', '/proc/net/dev' or return \%stats;
        while (<$fh>) {
            next unless /^\s*(\w+):\s*(\d+)\s+(\d+)\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+\d+\s+(\d+)\s+(\d+)/;
            my ($iface, $rx_bytes, $rx_packets, $tx_bytes, $tx_packets) =
                ($1, $2, $3, $4, $5);

            $stats{$iface} = {
                rx_bytes => $rx_bytes,
                rx_packets => $rx_packets,
                tx_bytes => $tx_bytes,
                tx_packets => $tx_packets,
            };
        }
        close $fh;
    } else {
        # Use netstat for other systems
        my @netstat = `netstat -ib 2>/dev/null`;
        for my $line (@netstat) {
            next unless $line =~ /^(\w+)\s+\d+\s+\S+\s+\S+\s+(\d+)\s+\d+\s+(\d+)/;
            $stats{$1} = {
                rx_bytes => $2,
                tx_bytes => $3,
            };
        }
    }

    return \%stats;
}

# Usage
package main;

my $monitor = SystemMonitor->new();

while (1) {
    my $cpu = $monitor->get_cpu_usage();
    my $mem = $monitor->get_memory_info();
    my @load = $monitor->get_load_average();
    my $disks = $monitor->get_disk_usage();
    my $net = $monitor->get_network_stats();

    print "\033[2J\033[H";  # Clear screen
    say "=" x 60;
    say "System Monitor - " . localtime();
    say "=" x 60;

    say "\nCPU Usage: $cpu%";
    say "Load Average: " . join(", ", @load);

    printf "\nMemory: %.1f GB / %.1f GB (%.1f%% used)\n",
        $mem->{used} / (1024**3),
        $mem->{total} / (1024**3),
        ($mem->{used} / $mem->{total}) * 100;

    say "\nDisk Usage:";
    for my $disk (@$disks) {
        printf "  %-20s %s (%.1f GB free)\n",
            $disk->{mount},
            $disk->{percent},
            $disk->{available} / (1024**3);
    }

    sleep 5;
}
```

## Service Health Checks

### Multi-Service Monitor

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use LWP::UserAgent;
use Net::Ping;
use DBI;
use Redis;
use IO::Socket::INET;
use Time::HiRes qw(time);

package ServiceMonitor;
use Moo;

has 'services' => (is => 'ro', default => sub { [] });
has 'ua' => (is => 'lazy');
has 'results' => (is => 'rw', default => sub { {} });

sub _build_ua {
    return LWP::UserAgent->new(
        timeout => 10,
        agent => 'ServiceMonitor/1.0',
    );
}

sub add_service {
    my ($self, %service) = @_;
    push @{$self->services}, \%service;
}

sub check_all {
    my $self = shift;

    for my $service (@{$self->services}) {
        my $result = $self->check_service($service);
        $self->results->{$service->{name}} = $result;
    }

    return $self->results;
}

sub check_service {
    my ($self, $service) = @_;

    my $start = time();
    my $result = {
        name => $service->{name},
        type => $service->{type},
        status => 'unknown',
        message => '',
        response_time => 0,
        checked_at => time(),
    };

    my $method = "check_" . $service->{type};
    if ($self->can($method)) {
        eval {
            $self->$method($service, $result);
        };
        if ($@) {
            $result->{status} = 'error';
            $result->{message} = $@;
        }
    } else {
        $result->{status} = 'error';
        $result->{message} = "Unknown service type: $service->{type}";
    }

    $result->{response_time} = sprintf("%.3f", time() - $start);
    return $result;
}

sub check_http {
    my ($self, $service, $result) = @_;

    my $response = $self->ua->get($service->{url});

    if ($response->is_success) {
        $result->{status} = 'up';

        # Check for expected content
        if ($service->{expected_content}) {
            if ($response->content =~ /$service->{expected_content}/) {
                $result->{message} = 'Content check passed';
            } else {
                $result->{status} = 'degraded';
                $result->{message} = 'Content check failed';
            }
        }

        # Check response time threshold
        if ($service->{max_response_time} &&
            $result->{response_time} > $service->{max_response_time}) {
            $result->{status} = 'degraded';
            $result->{message} = 'Slow response';
        }
    } else {
        $result->{status} = 'down';
        $result->{message} = $response->status_line;
    }
}

sub check_tcp {
    my ($self, $service, $result) = @_;

    my $socket = IO::Socket::INET->new(
        PeerAddr => $service->{host},
        PeerPort => $service->{port},
        Proto    => 'tcp',
        Timeout  => $service->{timeout} // 5,
    );

    if ($socket) {
        $result->{status} = 'up';
        $result->{message} = "Port $service->{port} is open";
        close $socket;
    } else {
        $result->{status} = 'down';
        $result->{message} = "Cannot connect to $service->{host}:$service->{port}";
    }
}

sub check_ping {
    my ($self, $service, $result) = @_;

    my $ping = Net::Ping->new('tcp', 2);
    $ping->port_number($service->{port}) if $service->{port};

    if ($ping->ping($service->{host})) {
        $result->{status} = 'up';
        $result->{message} = 'Host is reachable';
    } else {
        $result->{status} = 'down';
        $result->{message} = 'Host is unreachable';
    }

    $ping->close();
}

sub check_database {
    my ($self, $service, $result) = @_;

    my $dbh = DBI->connect(
        $service->{dsn},
        $service->{user},
        $service->{password},
        { RaiseError => 0, PrintError => 0 }
    );

    if ($dbh) {
        # Try a simple query
        my $test_query = $service->{test_query} // 'SELECT 1';
        if ($dbh->do($test_query)) {
            $result->{status} = 'up';
            $result->{message} = 'Database is responsive';
        } else {
            $result->{status} = 'degraded';
            $result->{message} = 'Database connected but query failed';
        }
        $dbh->disconnect;
    } else {
        $result->{status} = 'down';
        $result->{message} = $DBI::errstr;
    }
}

sub check_redis {
    my ($self, $service, $result) = @_;

    eval {
        my $redis = Redis->new(
            server => "$service->{host}:$service->{port}",
            reconnect => 2,
            read_timeout => 2,
        );

        $redis->ping;
        $result->{status} = 'up';

        # Check memory usage
        my $info = $redis->info('memory');
        if ($info =~ /used_memory:(\d+)/) {
            my $used = $1;
            $result->{message} = sprintf("Memory: %.1f MB", $used / 1024 / 1024);
        }

        $redis->quit;
    };

    if ($@) {
        $result->{status} = 'down';
        $result->{message} = $@;
    }
}

sub check_process {
    my ($self, $service, $result) = @_;

    my $process = $service->{process};
    my @pids = $self->find_process($process);

    if (@pids) {
        $result->{status} = 'up';
        $result->{message} = "Process running (PID: " . join(", ", @pids) . ")";

        # Check process count
        if ($service->{min_processes} && @pids < $service->{min_processes}) {
            $result->{status} = 'degraded';
            $result->{message} = "Only " . scalar(@pids) . " processes running";
        }
    } else {
        $result->{status} = 'down';
        $result->{message} = "Process not found: $process";
    }
}

sub find_process {
    my ($self, $name) = @_;
    my @pids;

    if ($^O eq 'linux') {
        @pids = map { /^\s*(\d+)/ ? $1 : () } `pgrep -f "$name"`;
    } else {
        my @ps = `ps aux`;
        for my $line (@ps) {
            if ($line =~ /^\S+\s+(\d+).*$name/ && $line !~ /grep/) {
                push @pids, $1;
            }
        }
    }

    return @pids;
}

# Usage example
package main;

my $monitor = ServiceMonitor->new();

# Add services to monitor
$monitor->add_service(
    name => 'Web Server',
    type => 'http',
    url => 'https://example.com/health',
    expected_content => 'OK',
    max_response_time => 2,
);

$monitor->add_service(
    name => 'Database',
    type => 'database',
    dsn => 'DBI:mysql:database=myapp;host=localhost',
    user => 'monitor',
    password => 'secret',
    test_query => 'SELECT COUNT(*) FROM users',
);

$monitor->add_service(
    name => 'Redis Cache',
    type => 'redis',
    host => 'localhost',
    port => 6379,
);

$monitor->add_service(
    name => 'SSH Server',
    type => 'tcp',
    host => 'localhost',
    port => 22,
);

$monitor->add_service(
    name => 'Nginx',
    type => 'process',
    process => 'nginx',
    min_processes => 2,
);

# Check all services
my $results = $monitor->check_all();

# Display results
for my $name (sort keys %$results) {
    my $r = $results->{$name};
    my $status_color = $r->{status} eq 'up' ? "\e[32m" :     # Green
                       $r->{status} eq 'degraded' ? "\e[33m" : # Yellow
                       "\e[31m";                                # Red

    printf "%s%-20s %s%-10s\e[0m %s (%.3fs)\n",
        $status_color,
        $name,
        $status_color,
        uc($r->{status}),
        $r->{message},
        $r->{response_time};
}
```

## Alerting Systems

### Multi-Channel Alerting

```perl
#!/usr/bin/env perl
package AlertManager;
use Modern::Perl '2023';
use Moo;
use Email::Simple;
use Email::Sender::Simple qw(sendmail);
use LWP::UserAgent;
use JSON::XS;

has 'config' => (is => 'ro', required => 1);
has 'alert_history' => (is => 'rw', default => sub { {} });
has 'ua' => (is => 'lazy');

sub _build_ua {
    return LWP::UserAgent->new(timeout => 10);
}

sub send_alert {
    my ($self, $alert) = @_;

    # Deduplicate alerts
    my $alert_key = "$alert->{service}:$alert->{severity}:$alert->{message}";
    my $last_sent = $self->alert_history->{$alert_key} // 0;

    # Rate limit: don't send same alert more than once per hour
    if (time() - $last_sent < 3600) {
        return;
    }

    # Send to configured channels based on severity
    my @channels = $self->get_channels_for_severity($alert->{severity});

    for my $channel (@channels) {
        my $method = "send_to_$channel";
        if ($self->can($method)) {
            eval { $self->$method($alert) };
            warn "Failed to send to $channel: $@" if $@;
        }
    }

    $self->alert_history->{$alert_key} = time();
}

sub get_channels_for_severity {
    my ($self, $severity) = @_;

    my %severity_channels = (
        critical => [qw(email sms slack pagerduty)],
        warning  => [qw(email slack)],
        info     => [qw(slack)],
    );

    return @{$severity_channels{$severity} // ['email']};
}

sub send_to_email {
    my ($self, $alert) = @_;

    my $email = Email::Simple->create(
        header => [
            To      => $self->config->{email}{to},
            From    => $self->config->{email}{from},
            Subject => "[$alert->{severity}] $alert->{service} Alert",
        ],
        body => $self->format_alert_text($alert),
    );

    sendmail($email);
}

sub send_to_slack {
    my ($self, $alert) = @_;

    my $color = $alert->{severity} eq 'critical' ? 'danger' :
                $alert->{severity} eq 'warning' ? 'warning' : 'good';

    my $payload = {
        channel => $self->config->{slack}{channel},
        username => 'Monitor Bot',
        attachments => [{
            color => $color,
            title => "$alert->{service} Alert",
            text => $alert->{message},
            fields => [
                {
                    title => 'Severity',
                    value => uc($alert->{severity}),
                    short => \1,
                },
                {
                    title => 'Time',
                    value => localtime($alert->{timestamp}),
                    short => \1,
                },
            ],
            footer => $alert->{host} // 'monitoring-system',
            ts => $alert->{timestamp},
        }],
    };

    $self->ua->post(
        $self->config->{slack}{webhook_url},
        Content_Type => 'application/json',
        Content => encode_json($payload),
    );
}

sub send_to_sms {
    my ($self, $alert) = @_;

    # Using Twilio API as example
    my $message = substr($self->format_alert_text($alert), 0, 160);

    my $response = $self->ua->post(
        "https://api.twilio.com/2010-04-01/Accounts/" .
        $self->config->{twilio}{account_sid} . "/Messages.json",
        Authorization => 'Basic ' . MIME::Base64::encode_base64(
            $self->config->{twilio}{account_sid} . ':' .
            $self->config->{twilio}{auth_token}
        ),
        Content => {
            From => $self->config->{twilio}{from_number},
            To   => $self->config->{twilio}{to_number},
            Body => $message,
        },
    );
}

sub send_to_pagerduty {
    my ($self, $alert) = @_;

    my $payload = {
        routing_key => $self->config->{pagerduty}{integration_key},
        event_action => 'trigger',
        dedup_key => "$alert->{service}:$alert->{check}",
        payload => {
            summary => "$alert->{service}: $alert->{message}",
            severity => $alert->{severity} eq 'critical' ? 'critical' : 'warning',
            source => $alert->{host} // 'monitoring',
            custom_details => $alert->{details} // {},
        },
    };

    $self->ua->post(
        'https://events.pagerduty.com/v2/enqueue',
        Content_Type => 'application/json',
        Content => encode_json($payload),
    );
}

sub format_alert_text {
    my ($self, $alert) = @_;

    my $text = <<END;
Alert: $alert->{service}
Severity: $alert->{severity}
Time: @{[scalar localtime($alert->{timestamp})]}
Host: @{[$alert->{host} // 'unknown']}

Message:
$alert->{message}

END

    if ($alert->{details}) {
        $text .= "Details:\n";
        for my $key (sort keys %{$alert->{details}}) {
            $text .= "  $key: $alert->{details}{$key}\n";
        }
    }

    return $text;
}
```

## Log Monitoring and Analysis

### Real-Time Log Monitor

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use File::Tail;
use Time::Piece;

package LogMonitor;
use Moo;

has 'config' => (is => 'ro', required => 1);
has 'patterns' => (is => 'rw', default => sub { [] });
has 'counters' => (is => 'rw', default => sub { {} });
has 'alert_manager' => (is => 'ro', required => 1);

sub add_pattern {
    my ($self, %pattern) = @_;
    push @{$self->patterns}, {
        name => $pattern{name},
        regex => qr/$pattern{regex}/,
        severity => $pattern{severity} // 'warning',
        threshold => $pattern{threshold} // 1,
        window => $pattern{window} // 60,  # seconds
        action => $pattern{action},
    };
}

sub monitor_file {
    my ($self, $file) = @_;

    my $tail = File::Tail->new(
        name => $file,
        interval => 1,
        maxinterval => 5,
        tail => 0,
    );

    while (defined(my $line = $tail->read)) {
        chomp $line;
        $self->process_line($line, $file);
    }
}

sub process_line {
    my ($self, $line, $file) = @_;

    for my $pattern (@{$self->patterns}) {
        if ($line =~ $pattern->{regex}) {
            $self->handle_match($pattern, $line, $file);
        }
    }

    # Update general statistics
    $self->update_stats($line);
}

sub handle_match {
    my ($self, $pattern, $line, $file) = @_;

    my $now = time();
    my $key = $pattern->{name};

    # Initialize counter
    $self->counters->{$key} //= {
        count => 0,
        timestamps => [],
        last_alert => 0,
    };

    my $counter = $self->counters->{$key};

    # Add to timestamps
    push @{$counter->{timestamps}}, $now;

    # Remove old timestamps outside window
    @{$counter->{timestamps}} = grep {
        $_ > $now - $pattern->{window}
    } @{$counter->{timestamps}};

    $counter->{count} = scalar @{$counter->{timestamps}};

    # Check threshold
    if ($counter->{count} >= $pattern->{threshold}) {
        # Rate limit alerts
        if ($now - $counter->{last_alert} > 300) {  # 5 minutes
            $self->send_alert($pattern, $line, $file, $counter->{count});
            $counter->{last_alert} = $now;

            # Execute action if defined
            if ($pattern->{action}) {
                $self->execute_action($pattern->{action}, $line);
            }
        }
    }
}

sub send_alert {
    my ($self, $pattern, $line, $file, $count) = @_;

    $self->alert_manager->send_alert({
        service => "Log Monitor - $file",
        severity => $pattern->{severity},
        message => "Pattern '$pattern->{name}' matched $count times in $pattern->{window} seconds",
        timestamp => time(),
        details => {
            pattern => $pattern->{name},
            last_match => $line,
            count => $count,
            file => $file,
        },
    });
}

sub execute_action {
    my ($self, $action, $line) = @_;

    if (ref $action eq 'CODE') {
        $action->($line);
    } elsif (!ref $action) {
        # Shell command
        system($action);
    }
}

sub update_stats {
    my ($self, $line) = @_;

    # Count log levels
    if ($line =~ /\[(ERROR|WARN|INFO|DEBUG)\]/) {
        $self->counters->{"level_$1"}++;
    }

    # Track response times
    if ($line =~ /response_time=(\d+)ms/) {
        push @{$self->counters->{response_times}}, $1;
    }
}

# Usage
package main;

my $alert_manager = AlertManager->new(
    config => {
        email => {
            to => 'admin@example.com',
            from => 'monitor@example.com',
        },
        slack => {
            webhook_url => 'https://hooks.slack.com/services/XXX',
            channel => '#alerts',
        },
    },
);

my $monitor = LogMonitor->new(
    config => {},
    alert_manager => $alert_manager,
);

# Add monitoring patterns
$monitor->add_pattern(
    name => 'Out of Memory',
    regex => 'Out of memory|OOM|memory exhausted',
    severity => 'critical',
    threshold => 1,
    action => sub {
        system('systemctl restart app.service');
        warn "Restarted application due to OOM";
    },
);

$monitor->add_pattern(
    name => 'High Error Rate',
    regex => 'ERROR|Exception|Fatal',
    severity => 'warning',
    threshold => 10,
    window => 60,
);

$monitor->add_pattern(
    name => 'Authentication Failures',
    regex => 'authentication failed|invalid password|login failed',
    severity => 'warning',
    threshold => 5,
    window => 300,
    action => sub {
        my $line = shift;
        if ($line =~ /from\s+(\S+)/) {
            my $ip = $1;
            system("iptables -A INPUT -s $ip -j DROP");
            warn "Blocked IP: $ip";
        }
    },
);

# Monitor multiple files
use threads;

my @files = qw(
    /var/log/app/application.log
    /var/log/nginx/error.log
    /var/log/mysql/error.log
);

my @threads;
for my $file (@files) {
    push @threads, threads->create(sub {
        $monitor->monitor_file($file);
    });
}

$_->join for @threads;
```

## Complete Monitoring System

### Integrated Monitoring Solution

```perl
#!/usr/bin/env perl
package MonitoringSystem;
use Modern::Perl '2023';
use Moo;
use YAML::XS qw(LoadFile);
use DBI;
use Time::HiRes qw(time sleep);
use threads;
use Thread::Queue;

has 'config_file' => (is => 'ro', required => 1);
has 'config' => (is => 'lazy');
has 'db' => (is => 'lazy');
has 'queue' => (is => 'ro', default => sub { Thread::Queue->new() });
has 'monitors' => (is => 'rw', default => sub { {} });

sub _build_config {
    my $self = shift;
    return LoadFile($self->config_file);
}

sub _build_db {
    my $self = shift;
    return DBI->connect(
        $self->config->{database}{dsn},
        $self->config->{database}{user},
        $self->config->{database}{password},
        { RaiseError => 1, AutoCommit => 1 }
    );
}

sub setup_database {
    my $self = shift;

    $self->db->do(<<'SQL');
CREATE TABLE IF NOT EXISTS metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    host TEXT NOT NULL,
    metric TEXT NOT NULL,
    value REAL,
    timestamp INTEGER NOT NULL,
    metadata TEXT
);
SQL

    $self->db->do(<<'SQL');
CREATE TABLE IF NOT EXISTS alerts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    service TEXT NOT NULL,
    severity TEXT NOT NULL,
    message TEXT NOT NULL,
    status TEXT DEFAULT 'open',
    created_at INTEGER NOT NULL,
    resolved_at INTEGER
);
SQL

    $self->db->do(<<'SQL');
CREATE INDEX IF NOT EXISTS idx_metrics_time ON metrics(timestamp);
SQL

    $self->db->do(<<'SQL');
CREATE INDEX IF NOT EXISTS idx_alerts_status ON alerts(status);
SQL
}

sub start {
    my $self = shift;

    $self->setup_database();

    # Start metric collectors
    $self->start_system_monitor();
    $self->start_service_monitor();
    $self->start_log_monitor();

    # Start metric processor
    $self->start_processor();

    # Start alert evaluator
    $self->start_alert_evaluator();

    # Wait for shutdown signal
    $SIG{INT} = $SIG{TERM} = sub {
        say "Shutting down...";
        $self->queue->end();
        exit 0;
    };

    while (1) {
        sleep 60;
        $self->cleanup_old_data();
    }
}

sub start_system_monitor {
    my $self = shift;

    threads->create(sub {
        my $monitor = SystemMonitor->new();

        while (1) {
            my $metrics = {
                cpu => $monitor->get_cpu_usage(),
                memory => $monitor->get_memory_info(),
                disk => $monitor->get_disk_usage(),
                load => [$monitor->get_load_average()],
            };

            $self->queue->enqueue({
                type => 'system',
                host => hostname(),
                metrics => $metrics,
                timestamp => time(),
            });

            sleep $self->config->{intervals}{system} // 60;
        }
    })->detach();
}

sub start_processor {
    my $self = shift;

    threads->create(sub {
        while (my $item = $self->queue->dequeue()) {
            $self->process_metric($item);
        }
    })->detach();
}

sub process_metric {
    my ($self, $item) = @_;

    if ($item->{type} eq 'system') {
        # Store CPU metric
        $self->store_metric(
            $item->{host},
            'cpu_usage',
            $item->{metrics}{cpu},
            $item->{timestamp}
        );

        # Store memory metrics
        my $mem = $item->{metrics}{memory};
        my $mem_percent = ($mem->{used} / $mem->{total}) * 100;
        $self->store_metric(
            $item->{host},
            'memory_percent',
            $mem_percent,
            $item->{timestamp}
        );

        # Store disk metrics
        for my $disk (@{$item->{metrics}{disk}}) {
            my $percent = $disk->{percent};
            $percent =~ s/%//;
            $self->store_metric(
                $item->{host},
                "disk_usage_$disk->{mount}",
                $percent,
                $item->{timestamp},
                { filesystem => $disk->{filesystem} }
            );
        }
    }
}

sub store_metric {
    my ($self, $host, $metric, $value, $timestamp, $metadata) = @_;

    $self->db->do(
        "INSERT INTO metrics (host, metric, value, timestamp, metadata) VALUES (?, ?, ?, ?, ?)",
        undef,
        $host,
        $metric,
        $value,
        $timestamp,
        $metadata ? encode_json($metadata) : undef
    );
}

sub start_alert_evaluator {
    my $self = shift;

    threads->create(sub {
        while (1) {
            $self->evaluate_alerts();
            sleep 30;
        }
    })->detach();
}

sub evaluate_alerts {
    my $self = shift;

    for my $rule (@{$self->config->{alert_rules}}) {
        my $result = $self->evaluate_rule($rule);

        if ($result->{triggered}) {
            $self->create_alert($rule, $result);
        }
    }
}

sub evaluate_rule {
    my ($self, $rule) = @_;

    # Get recent metrics
    my $sth = $self->db->prepare(
        "SELECT AVG(value) as avg_value FROM metrics
         WHERE metric = ? AND timestamp > ?
         GROUP BY host HAVING avg_value > ?"
    );

    my $window_start = time() - ($rule->{window} // 300);
    $sth->execute($rule->{metric}, $window_start, $rule->{threshold});

    my $triggered = $sth->fetchrow_hashref() ? 1 : 0;

    return {
        triggered => $triggered,
        metric => $rule->{metric},
        threshold => $rule->{threshold},
    };
}

sub cleanup_old_data {
    my $self = shift;

    my $retention = $self->config->{retention} // 7 * 24 * 3600;  # 7 days
    my $cutoff = time() - $retention;

    $self->db->do("DELETE FROM metrics WHERE timestamp < ?", undef, $cutoff);
    $self->db->do("DELETE FROM alerts WHERE created_at < ? AND status = 'resolved'",
                  undef, $cutoff);
}

sub hostname {
    my $hostname = `hostname`;
    chomp $hostname;
    return $hostname;
}

# Main execution
package main;

unless (caller) {
    my $system = MonitoringSystem->new(
        config_file => 'monitoring.yaml',
    );

    $system->start();
}
```

## Best Practices

1. **Monitor what matters** - Focus on user-facing metrics
2. **Set appropriate thresholds** - Avoid alert fatigue
3. **Use multiple data points** - Don't alert on single spikes
4. **Implement alert deduplication** - Prevent spam
5. **Keep historical data** - For trend analysis
6. **Monitor the monitors** - Ensure monitoring is working
7. **Document alert responses** - Create runbooks
8. **Test alert channels** - Ensure alerts get through
9. **Use structured logging** - Makes parsing easier
10. **Implement graceful degradation** - Don't crash on monitoring failures

## Conclusion

Effective monitoring is about more than just checking if services are up. It's about understanding system behavior, preventing problems, and responding quickly when issues occur. Perl's system access capabilities and text processing power make it ideal for building comprehensive monitoring solutions.

Remember: monitoring is not a set-and-forget activity. Continuously refine your thresholds, add new checks as your system evolves, and always be ready to respond to the unexpected.

---

*Next: Automation workflows and cron jobs. We'll explore how to build reliable automation that runs on schedule and handles complex workflows.*