# Chapter 9: Log File Analysis and Monitoring

> "The difference between a good sysadmin and a great one? The great one automated their log analysis before the outage." - Anonymous

Logs are the heartbeat of your systems. They tell you what happened, when it happened, and sometimes even why. But with modern systems generating gigabytes of logs daily, manual analysis is impossible. This chapter shows you how to build log analysis tools that find needles in haystacks, detect anomalies, and alert you before things go wrong.

## Understanding Log Formats

### Common Log Formats

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

# Apache Combined Log Format
sub parse_apache_log($line) {
    my $regex = qr/
        ^(\S+)\s+                    # IP address
        (\S+)\s+                     # Identity
        (\S+)\s+                     # User
        \[([^\]]+)\]\s+              # Timestamp
        "([^"]+)"\s+                 # Request
        (\d{3})\s+                   # Status code
        (\d+|-)\s*                   # Size
        "([^"]*)"\s*                 # Referrer
        "([^"]*)"                    # User agent
    /x;

    if ($line =~ $regex) {
        my ($method, $path, $protocol) = split /\s+/, $5;
        return {
            ip => $1,
            identity => $2 eq '-' ? undef : $2,
            user => $3 eq '-' ? undef : $3,
            timestamp => $4,
            method => $method,
            path => $path,
            protocol => $protocol,
            status => $6,
            size => $7 eq '-' ? 0 : $7,
            referrer => $8 eq '-' ? undef : $8,
            user_agent => $9,
        };
    }
    return undef;
}

# Syslog Format
sub parse_syslog($line) {
    my $regex = qr/
        ^(\w+\s+\d+\s+\d{2}:\d{2}:\d{2})\s+  # Timestamp
        (\S+)\s+                               # Hostname
        ([^:\[]+)                              # Program
        (?:\[(\d+)\])?                        # PID (optional)
        :\s*                                   # Separator
        (.+)$                                  # Message
    /x;

    if ($line =~ $regex) {
        return {
            timestamp => $1,
            hostname => $2,
            program => $3,
            pid => $4,
            message => $5,
        };
    }
    return undef;
}

# JSON Logs (structured logging)
use JSON::XS;

sub parse_json_log($line) {
    my $json = JSON::XS->new->utf8;

    eval {
        return $json->decode($line);
    };
    if ($@) {
        warn "Failed to parse JSON log: $@";
        return undef;
    }
}

# Custom Application Logs
sub parse_custom_log($line, $pattern) {
    if ($line =~ $pattern) {
        return { %+ };  # Return named captures
    }
    return undef;
}
```

## Real-Time Log Monitoring

### Tail Follow Implementation

```perl
use File::Tail;
use IO::Select;

# Monitor single file
sub tail_file {
    my ($filename, $callback) = @_;

    my $file = File::Tail->new(
        name => $filename,
        interval => 1,
        maxinterval => 5,
        adjustafter => 10,
        resetafter => 30,
        tail => 0,  # Start from end of file
    );

    while (defined(my $line = $file->read)) {
        chomp $line;
        $callback->($line);
    }
}

# Monitor multiple files
sub tail_multiple {
    my ($files, $callback) = @_;

    my @tailfiles = map {
        File::Tail->new(
            name => $_,
            interval => 1,
            tail => 0,
        )
    } @$files;

    while (1) {
        my ($nfound, $timeleft, @pending) =
            File::Tail::select(undef, undef, undef, 10, @tailfiles);

        foreach my $file (@pending) {
            my $line = $file->read;
            chomp $line;
            $callback->($file->{name}, $line);
        }
    }
}

# Real-time log processor
sub monitor_logs {
    my ($logfile) = @_;

    tail_file($logfile, sub {
        my ($line) = @_;

        # Parse log line
        my $entry = parse_apache_log($line);
        return unless $entry;

        # Check for errors
        if ($entry->{status} >= 500) {
            alert("Server error: $entry->{path} returned $entry->{status}");
        }

        # Check for attacks
        if ($entry->{path} =~ /\.\.[\/\\]|<script>|union\s+select/i) {
            alert("Possible attack from $entry->{ip}: $entry->{path}");
        }

        # Track metrics
        update_metrics($entry);
    });
}

sub alert {
    my ($message) = @_;
    say "[ALERT] " . localtime() . " - $message";
    # Could also send email, Slack message, etc.
}
```

## Log Analysis Patterns

### Error Detection and Classification

```perl
package LogAnalyzer;
use Modern::Perl '2023';

sub new {
    my ($class) = @_;
    return bless {
        patterns => [],
        stats => {},
        alerts => [],
    }, $class;
}

sub add_pattern {
    my ($self, $name, $regex, $severity) = @_;
    push @{$self->{patterns}}, {
        name => $name,
        regex => qr/$regex/i,
        severity => $severity,
        count => 0,
    };
}

sub analyze_line {
    my ($self, $line) = @_;

    for my $pattern (@{$self->{patterns}}) {
        if ($line =~ $pattern->{regex}) {
            $pattern->{count}++;
            $self->handle_match($pattern, $line);
            last;  # Only match first pattern
        }
    }
}

sub handle_match {
    my ($self, $pattern, $line) = @_;

    # Track statistics
    $self->{stats}{$pattern->{name}}++;
    $self->{stats}{by_severity}{$pattern->{severity}}++;

    # Generate alerts for high severity
    if ($pattern->{severity} >= 3) {
        push @{$self->{alerts}}, {
            timestamp => time(),
            pattern => $pattern->{name},
            severity => $pattern->{severity},
            line => $line,
        };
    }
}

sub get_summary {
    my ($self) = @_;
    return {
        patterns => [
            map { {
                name => $_->{name},
                count => $_->{count},
                severity => $_->{severity},
            } } @{$self->{patterns}}
        ],
        stats => $self->{stats},
        alerts => $self->{alerts},
    };
}

# Usage
my $analyzer = LogAnalyzer->new();

# Define patterns
$analyzer->add_pattern('out_of_memory', 'out of memory|OOM', 5);
$analyzer->add_pattern('disk_full', 'no space left|disk full', 4);
$analyzer->add_pattern('connection_failed', 'connection refused|timeout', 3);
$analyzer->add_pattern('auth_failure', 'authentication failed|invalid password', 2);
$analyzer->add_pattern('not_found', '404|not found', 1);

# Process logs
open my $fh, '<', 'application.log' or die $!;
while (<$fh>) {
    $analyzer->analyze_line($_);
}
close $fh;

# Get results
my $summary = $analyzer->get_summary();
```

### Anomaly Detection

```perl
# Statistical anomaly detection
package AnomalyDetector;
use Statistics::Descriptive;

sub new {
    my ($class, $window_size) = @_;
    return bless {
        window_size => $window_size // 100,
        values => [],
        stats => Statistics::Descriptive::Full->new(),
    }, $class;
}

sub add_value {
    my ($self, $value) = @_;

    push @{$self->{values}}, $value;

    # Maintain window
    if (@{$self->{values}} > $self->{window_size}) {
        shift @{$self->{values}};
    }

    # Update statistics
    $self->{stats}->clear();
    $self->{stats}->add_data(@{$self->{values}});
}

sub is_anomaly {
    my ($self, $value, $threshold) = @_;
    $threshold //= 3;  # Default to 3 standard deviations

    return 0 if @{$self->{values}} < 10;  # Need minimum data

    my $mean = $self->{stats}->mean();
    my $stddev = $self->{stats}->standard_deviation();

    return 0 if $stddev == 0;  # No variation

    my $z_score = abs(($value - $mean) / $stddev);
    return $z_score > $threshold;
}

# Detect unusual request rates
sub monitor_request_rate {
    my ($logfile) = @_;

    my $detector = AnomalyDetector->new(60);  # 60-minute window
    my %requests_per_minute;

    tail_file($logfile, sub {
        my ($line) = @_;
        my $entry = parse_apache_log($line);
        return unless $entry;

        # Count requests per minute
        my $minute = substr($entry->{timestamp}, 0, 16);  # Trim seconds
        $requests_per_minute{$minute}++;

        # Check for anomalies every minute
        state $last_minute = '';
        if ($minute ne $last_minute && $last_minute) {
            my $count = $requests_per_minute{$last_minute} // 0;
            $detector->add_value($count);

            if ($detector->is_anomaly($count)) {
                alert("Anomaly detected: $count requests in $last_minute");
            }

            delete $requests_per_minute{$last_minute};
        }
        $last_minute = $minute;
    });
}
```

## Log Aggregation and Reporting

### Multi-File Log Aggregation

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use File::Find;
use IO::Uncompress::Gunzip qw(gunzip);

# Aggregate logs from multiple sources
sub aggregate_logs {
    my ($log_dirs, $pattern, $start_time, $end_time) = @_;

    my @entries;

    for my $dir (@$log_dirs) {
        find(sub {
            return unless /$pattern/;

            my $file = $File::Find::name;
            push @entries, @{process_log_file($file, $start_time, $end_time)};
        }, $dir);
    }

    # Sort by timestamp
    @entries = sort { $a->{timestamp} cmp $b->{timestamp} } @entries;

    return \@entries;
}

sub process_log_file {
    my ($file, $start_time, $end_time) = @_;
    my @entries;

    my $fh;
    if ($file =~ /\.gz$/) {
        $fh = IO::Uncompress::Gunzip->new($file) or die "Can't open $file: $!";
    } else {
        open $fh, '<', $file or die "Can't open $file: $!";
    }

    while (my $line = <$fh>) {
        chomp $line;
        my $entry = parse_log_line($line);
        next unless $entry;

        # Filter by time range
        next if $start_time && $entry->{timestamp} lt $start_time;
        next if $end_time && $entry->{timestamp} gt $end_time;

        push @entries, $entry;
    }

    close $fh;
    return \@entries;
}
```

### Report Generation

```perl
# Generate HTML report
sub generate_html_report {
    my ($data, $output_file) = @_;

    open my $fh, '>:encoding(utf8)', $output_file or die $!;

    print $fh <<'HTML';
<!DOCTYPE html>
<html>
<head>
    <title>Log Analysis Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #4CAF50; color: white; }
        tr:nth-child(even) { background-color: #f2f2f2; }
        .error { color: red; font-weight: bold; }
        .warning { color: orange; }
        .chart { margin: 20px 0; }
    </style>
</head>
<body>
    <h1>Log Analysis Report</h1>
HTML

    # Summary section
    print $fh "<h2>Summary</h2>\n";
    print $fh "<ul>\n";
    print $fh "<li>Total Entries: $data->{total}</li>\n";
    print $fh "<li>Time Range: $data->{start_time} to $data->{end_time}</li>\n";
    print $fh "<li>Errors: $data->{error_count}</li>\n";
    print $fh "<li>Warnings: $data->{warning_count}</li>\n";
    print $fh "</ul>\n";

    # Top errors table
    print $fh "<h2>Top Errors</h2>\n";
    print $fh "<table>\n";
    print $fh "<tr><th>Error</th><th>Count</th><th>Percentage</th></tr>\n";

    for my $error (@{$data->{top_errors}}) {
        my $percentage = sprintf("%.2f%%",
            ($error->{count} / $data->{error_count}) * 100);
        print $fh "<tr><td>$error->{message}</td>";
        print $fh "<td>$error->{count}</td>";
        print $fh "<td>$percentage</td></tr>\n";
    }

    print $fh "</table>\n";

    # Timeline chart (ASCII)
    print $fh "<h2>Activity Timeline</h2>\n";
    print $fh "<pre class='chart'>\n";
    print $fh generate_ascii_chart($data->{timeline});
    print $fh "</pre>\n";

    print $fh "</body></html>\n";
    close $fh;

    say "Report generated: $output_file";
}

sub generate_ascii_chart {
    my ($timeline) = @_;

    my $max_value = 0;
    for my $point (@$timeline) {
        $max_value = $point->{value} if $point->{value} > $max_value;
    }

    my $chart = "";
    my $scale = 50 / ($max_value || 1);

    for my $point (@$timeline) {
        my $bar_length = int($point->{value} * $scale);
        my $bar = '#' x $bar_length;
        $chart .= sprintf("%s | %-50s %d\n",
            $point->{time}, $bar, $point->{value});
    }

    return $chart;
}
```

## Advanced Log Processing

### Pattern Mining

```perl
# Discover common patterns in logs
sub mine_patterns {
    my ($logfile, $min_support) = @_;
    $min_support //= 10;

    my %patterns;
    my $total_lines = 0;

    open my $fh, '<', $logfile or die $!;
    while (my $line = <$fh>) {
        chomp $line;
        $total_lines++;

        # Tokenize line
        my @tokens = $line =~ /\S+/g;

        # Generate n-grams
        for my $n (2..5) {
            for (my $i = 0; $i <= @tokens - $n; $i++) {
                my $pattern = join(' ', @tokens[$i..$i+$n-1]);

                # Replace variables with placeholders
                $pattern =~ s/\d+/NUM/g;
                $pattern =~ s/\b[A-Fa-f0-9]{32,}\b/HASH/g;
                $pattern =~ s/\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b/IP/g;

                $patterns{$pattern}++;
            }
        }
    }
    close $fh;

    # Filter by support
    my @frequent = grep { $patterns{$_} >= $min_support } keys %patterns;
    @frequent = sort { $patterns{$b} <=> $patterns{$a} } @frequent;

    return \@frequent[0..min(99, $#frequent)];  # Top 100
}

sub min { $_[0] < $_[1] ? $_[0] : $_[1] }
```

### Session Reconstruction

```perl
# Reconstruct user sessions from logs
sub reconstruct_sessions {
    my ($logfile, $session_timeout) = @_;
    $session_timeout //= 1800;  # 30 minutes

    my %sessions;

    open my $fh, '<', $logfile or die $!;
    while (my $line = <$fh>) {
        my $entry = parse_apache_log($line);
        next unless $entry;

        my $session_id = identify_session($entry);
        next unless $session_id;

        # Check if session exists and is still active
        if (exists $sessions{$session_id}) {
            my $last_time = $sessions{$session_id}{last_activity};
            my $current_time = parse_timestamp($entry->{timestamp});

            if ($current_time - $last_time > $session_timeout) {
                # Session expired, start new one
                finalize_session($sessions{$session_id});
                delete $sessions{$session_id};
            }
        }

        # Add to session
        $sessions{$session_id} //= {
            id => $session_id,
            start_time => $entry->{timestamp},
            ip => $entry->{ip},
            user_agent => $entry->{user_agent},
            requests => [],
        };

        push @{$sessions{$session_id}{requests}}, {
            timestamp => $entry->{timestamp},
            method => $entry->{method},
            path => $entry->{path},
            status => $entry->{status},
            size => $entry->{size},
        };

        $sessions{$session_id}{last_activity} =
            parse_timestamp($entry->{timestamp});
    }
    close $fh;

    # Finalize remaining sessions
    finalize_session($_) for values %sessions;

    return \%sessions;
}

sub identify_session {
    my ($entry) = @_;

    # Look for session ID in various places
    if ($entry->{path} =~ /[?&]session=([^&]+)/) {
        return $1;
    }

    # Use IP + User Agent as fallback
    return "$entry->{ip}::" . ($entry->{user_agent} // 'unknown');
}

sub finalize_session {
    my ($session) = @_;

    $session->{duration} =
        parse_timestamp($session->{requests}[-1]{timestamp}) -
        parse_timestamp($session->{requests}[0]{timestamp});

    $session->{request_count} = scalar(@{$session->{requests}});
    $session->{total_bytes} = 0;
    $session->{error_count} = 0;

    for my $req (@{$session->{requests}}) {
        $session->{total_bytes} += $req->{size} // 0;
        $session->{error_count}++ if $req->{status} >= 400;
    }
}

sub parse_timestamp {
    # Convert log timestamp to epoch time
    # Implementation depends on log format
    my ($timestamp) = @_;
    # ... parsing logic ...
    return time();  # Placeholder
}
```

## Real-World Example: Complete Log Monitor

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use File::Tail;
use Email::Simple;
use Email::Sender::Simple qw(sendmail);
use DBI;

# Complete log monitoring system
package LogMonitor;

sub new {
    my ($class, $config) = @_;

    my $self = bless {
        config => $config,
        rules => [],
        metrics => {},
        alerts_sent => {},
    }, $class;

    $self->load_rules();
    $self->init_database() if $config->{use_database};

    return $self;
}

sub load_rules {
    my ($self) = @_;

    # Load monitoring rules from config
    for my $rule (@{$self->{config}{rules}}) {
        push @{$self->{rules}}, {
            name => $rule->{name},
            pattern => qr/$rule->{pattern}/,
            severity => $rule->{severity},
            threshold => $rule->{threshold} // 1,
            window => $rule->{window} // 60,
            action => $rule->{action},
            count => 0,
            timestamps => [],
        };
    }
}

sub process_line {
    my ($self, $line) = @_;

    for my $rule (@{$self->{rules}}) {
        if ($line =~ $rule->{pattern}) {
            $self->handle_rule_match($rule, $line);
        }
    }

    $self->update_metrics($line);
}

sub handle_rule_match {
    my ($self, $rule, $line) = @_;

    my $now = time();
    push @{$rule->{timestamps}}, $now;

    # Clean old timestamps
    @{$rule->{timestamps}} = grep {
        $_ > $now - $rule->{window}
    } @{$rule->{timestamps}};

    # Check threshold
    if (@{$rule->{timestamps}} >= $rule->{threshold}) {
        $self->trigger_alert($rule, $line);
    }
}

sub trigger_alert {
    my ($self, $rule, $line) = @_;

    # Rate limit alerts
    my $key = "$rule->{name}:" . int(time() / 300);  # 5-minute buckets
    return if $self->{alerts_sent}{$key}++;

    say "[ALERT] $rule->{name}: $line";

    # Execute action
    if ($rule->{action} eq 'email') {
        $self->send_email_alert($rule, $line);
    } elsif ($rule->{action} eq 'script') {
        system($rule->{script}, $rule->{name}, $line);
    }

    # Store in database
    if ($self->{config}{use_database}) {
        $self->store_alert($rule, $line);
    }
}

sub send_email_alert {
    my ($self, $rule, $line) = @_;

    my $email = Email::Simple->create(
        header => [
            To => $self->{config}{alert_email},
            From => 'logmonitor@example.com',
            Subject => "Alert: $rule->{name}",
        ],
        body => "Alert triggered: $rule->{name}\n\n" .
                "Pattern: $rule->{pattern}\n" .
                "Severity: $rule->{severity}\n" .
                "Log line: $line\n",
    );

    eval { sendmail($email) };
    warn "Failed to send email: $@" if $@;
}

sub update_metrics {
    my ($self, $line) = @_;

    $self->{metrics}{lines_processed}++;

    # Extract and track custom metrics
    if ($line =~ /response_time=(\d+)ms/) {
        push @{$self->{metrics}{response_times}}, $1;
    }

    # Periodic metric reporting
    if ($self->{metrics}{lines_processed} % 1000 == 0) {
        $self->report_metrics();
    }
}

sub report_metrics {
    my ($self) = @_;

    say "Metrics Report:";
    say "  Lines processed: $self->{metrics}{lines_processed}";

    if (my $times = $self->{metrics}{response_times}) {
        my $avg = sum(@$times) / @$times;
        say "  Avg response time: ${avg}ms";
        @$times = ();  # Clear for next period
    }
}

sub sum { my $s = 0; $s += $_ for @_; $s }

package main;

# Configuration
my $config = {
    use_database => 0,
    alert_email => 'admin@example.com',
    rules => [
        {
            name => 'High Error Rate',
            pattern => 'ERROR|FATAL',
            severity => 5,
            threshold => 10,
            window => 60,
            action => 'email',
        },
        {
            name => 'Security Alert',
            pattern => 'authentication failed|unauthorized access',
            severity => 4,
            threshold => 3,
            window => 300,
            action => 'script',
            script => '/usr/local/bin/security_response.sh',
        },
    ],
};

# Start monitoring
my $monitor = LogMonitor->new($config);

tail_file('/var/log/application.log', sub {
    my ($line) = @_;
    $monitor->process_line($line);
});
```

## Performance Tips

1. **Use compiled regexes** - Pre-compile patterns with qr//
2. **Process incrementally** - Don't load entire log into memory
3. **Index strategically** - Use seek/tell for random access
4. **Compress old logs** - Process gzipped files directly
5. **Parallel processing** - Use fork or threads for multiple files
6. **Cache parsed results** - Don't re-parse the same data
7. **Use appropriate data structures** - Hashes for lookups, arrays for order

## Best Practices

1. **Handle malformed lines gracefully** - Logs aren't always perfect
2. **Use time windows for analysis** - Avoid memory leaks
3. **Implement rate limiting for alerts** - Prevent alert storms
4. **Store raw logs** - Never modify originals
5. **Document log formats** - Include examples and edge cases
6. **Test with production data** - Synthetic logs miss real issues
7. **Monitor the monitor** - Ensure your tools are working

## Conclusion

Log analysis is where Perl's text processing power truly shines. Whether you're tracking down bugs, detecting security threats, or monitoring performance, Perl gives you the tools to turn log chaos into actionable insights. The key is building robust, efficient parsers and knowing when to alert versus when to aggregate.

Remember: logs are your system's story. Perl helps you read between the lines.

---

*Next: Process management and system commands. We'll explore how Perl interfaces with the operating system, manages processes, and automates system administration tasks.*