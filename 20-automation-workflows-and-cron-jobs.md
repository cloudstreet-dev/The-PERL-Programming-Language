# Chapter 20: Automation Workflows and Cron Jobs

> "Automation is not about replacing humans, it's about amplifying human capabilities and eliminating toil." - SRE Handbook

Automation is the heart of efficient system administration. Whether it's scheduled maintenance, data processing pipelines, or complex deployment workflows, Perl excels at tying systems together and making them work in harmony. This chapter covers everything from simple cron jobs to sophisticated workflow engines, helping you build automation that's reliable, maintainable, and scalable.

## Understanding Cron

### Cron Fundamentals

```perl
#!/usr/bin/env perl
# Understanding cron syntax and best practices

# Cron format:
# * * * * * command
# | | | | |
# | | | | +-- Day of week (0-7, Sunday = 0 or 7)
# | | | +---- Month (1-12)
# | | +------ Day of month (1-31)
# | +-------- Hour (0-23)
# +---------- Minute (0-59)

# Common patterns:
# 0 2 * * *         Daily at 2:00 AM
# */15 * * * *      Every 15 minutes
# 0 0 * * 0         Weekly on Sunday at midnight
# 0 0 1 * *         Monthly on the 1st at midnight
# 0 0 1 1 *         Yearly on January 1st at midnight
# 0 9-17 * * 1-5    Every hour from 9 AM to 5 PM on weekdays

# Cron environment differs from interactive shell
# Always use absolute paths and set environment

package CronJob;
use Modern::Perl '2023';
use File::Basename;
use Cwd 'abs_path';

sub setup_environment {
    # Set up environment for cron execution
    $ENV{PATH} = '/usr/local/bin:/usr/bin:/bin';
    $ENV{LANG} = 'en_US.UTF-8';

    # Change to script directory
    my $script_dir = dirname(abs_path($0));
    chdir $script_dir or die "Can't change to $script_dir: $!";

    # Ensure output is unbuffered
    $| = 1;

    # Set up logging
    my $log_dir = '/var/log/cronjobs';
    mkdir $log_dir unless -d $log_dir;

    my $log_file = "$log_dir/" . basename($0) . ".log";
    open(STDOUT, '>>', $log_file) or die "Can't open log: $!";
    open(STDERR, '>&', \*STDOUT) or die "Can't redirect STDERR: $!";

    say "=" x 60;
    say "Job started at " . localtime();
    say "=" x 60;
}

sub acquire_lock {
    my ($lock_file) = @_;

    # Prevent multiple instances
    use Fcntl qw(:flock);

    open my $lock_fh, '>', $lock_file or die "Can't create lock: $!";

    unless (flock($lock_fh, LOCK_EX | LOCK_NB)) {
        die "Another instance is already running";
    }

    # Write PID to lock file
    print $lock_fh $$;

    return $lock_fh;  # Keep file handle open to maintain lock
}

sub run {
    my ($job_name, $code) = @_;

    setup_environment();

    my $lock_file = "/var/run/$job_name.lock";
    my $lock = eval { acquire_lock($lock_file) };

    if ($@) {
        say "Failed to acquire lock: $@";
        exit 1;
    }

    eval {
        $code->();
    };

    if ($@) {
        say "Job failed: $@";
        send_alert("Cron job $job_name failed: $@");
        exit 1;
    }

    say "Job completed successfully at " . localtime();
    unlink $lock_file;
}

sub send_alert {
    my ($message) = @_;
    # Send email or other notification
    system("echo '$message' | mail -s 'Cron Job Alert' admin@example.com");
}

# Example usage:
CronJob::run('daily_backup', sub {
    # Your job logic here
    system('backup_script.sh');
});
```

### Managing Cron Jobs Programmatically

```perl
#!/usr/bin/env perl
package CronManager;
use Modern::Perl '2023';
use Config::Crontab;

sub new {
    my $class = shift;
    my $self = {
        crontab => Config::Crontab->new(),
    };

    $self->{crontab}->read;
    return bless $self, $class;
}

sub add_job {
    my ($self, %job) = @_;

    # Create new cron event
    my $event = Config::Crontab::Event->new(
        -minute  => $job{minute} // '*',
        -hour    => $job{hour} // '*',
        -dom     => $job{day_of_month} // '*',
        -month   => $job{month} // '*',
        -dow     => $job{day_of_week} // '*',
        -command => $job{command},
    );

    # Add comment if provided
    if ($job{comment}) {
        my $comment = Config::Crontab::Comment->new(
            -data => "# $job{comment}"
        );
        $self->{crontab}->last(
            Config::Crontab::Block->new(
                -lines => [$comment, $event]
            )
        );
    } else {
        $self->{crontab}->last($event);
    }

    $self->save();
}

sub remove_job {
    my ($self, $pattern) = @_;

    my @blocks = $self->{crontab}->blocks;
    my @keep;

    for my $block (@blocks) {
        if ($block->isa('Config::Crontab::Event')) {
            next if $block->command =~ /$pattern/;
        }
        push @keep, $block;
    }

    $self->{crontab}->blocks(@keep);
    $self->save();
}

sub list_jobs {
    my $self = shift;
    my @jobs;

    for my $block ($self->{crontab}->blocks) {
        next unless $block->isa('Config::Crontab::Event');

        push @jobs, {
            minute => $block->minute,
            hour => $block->hour,
            day_of_month => $block->dom,
            month => $block->month,
            day_of_week => $block->dow,
            command => $block->command,
        };
    }

    return \@jobs;
}

sub save {
    my $self = shift;
    $self->{crontab}->write;
}

# Example: Dynamic cron job management
my $cron = CronManager->new();

# Add a backup job
$cron->add_job(
    comment => 'Daily database backup',
    hour => 2,
    minute => 30,
    command => '/usr/local/bin/backup_db.pl',
);

# Add a cleanup job
$cron->add_job(
    comment => 'Clean old logs every Sunday',
    day_of_week => 0,
    hour => 3,
    minute => 0,
    command => '/usr/local/bin/cleanup_logs.pl --days=30',
);

# List all jobs
my $jobs = $cron->list_jobs();
for my $job (@$jobs) {
    say "Job: $job->{command}";
    say "  Schedule: $job->{minute} $job->{hour} $job->{day_of_month} " .
        "$job->{month} $job->{day_of_week}";
}
```

## Building Workflow Engines

### Simple Workflow System

```perl
#!/usr/bin/env perl
package Workflow;
use Modern::Perl '2023';
use Moo;
use Types::Standard qw(ArrayRef HashRef CodeRef);

has 'name' => (is => 'ro', required => 1);
has 'steps' => (is => 'ro', isa => ArrayRef, default => sub { [] });
has 'context' => (is => 'rw', isa => HashRef, default => sub { {} });
has 'on_error' => (is => 'rw', isa => CodeRef);
has 'on_success' => (is => 'rw', isa => CodeRef);
has 'status' => (is => 'rw', default => 'pending');
has 'current_step' => (is => 'rw', default => 0);

sub add_step {
    my ($self, $name, $handler, %options) = @_;

    push @{$self->steps}, {
        name => $name,
        handler => $handler,
        retry_count => $options{retry} // 0,
        timeout => $options{timeout},
        on_error => $options{on_error},
        rollback => $options{rollback},
    };

    return $self;
}

sub execute {
    my $self = shift;

    $self->status('running');
    say "Starting workflow: " . $self->name;

    for (my $i = 0; $i < @{$self->steps}; $i++) {
        $self->current_step($i);
        my $step = $self->steps->[$i];

        unless ($self->execute_step($step)) {
            $self->status('failed');
            $self->rollback($i);
            $self->on_error->($self) if $self->on_error;
            return 0;
        }
    }

    $self->status('completed');
    $self->on_success->($self) if $self->on_success;
    say "Workflow completed successfully";

    return 1;
}

sub execute_step {
    my ($self, $step) = @_;

    say "  Executing step: $step->{name}";

    my $attempts = 0;
    my $max_attempts = $step->{retry_count} + 1;

    while ($attempts < $max_attempts) {
        $attempts++;

        eval {
            if ($step->{timeout}) {
                local $SIG{ALRM} = sub { die "timeout\n" };
                alarm($step->{timeout});
            }

            $step->{handler}->($self->context);

            alarm(0) if $step->{timeout};
        };

        if ($@) {
            my $error = $@;
            chomp $error;

            if ($attempts < $max_attempts) {
                warn "    Step failed (attempt $attempts): $error";
                sleep(2 ** $attempts);  # Exponential backoff
            } else {
                say "    Step failed after $attempts attempts: $error";
                $step->{on_error}->($self->context, $error) if $step->{on_error};
                return 0;
            }
        } else {
            say "    Step completed successfully";
            return 1;
        }
    }

    return 0;
}

sub rollback {
    my ($self, $failed_step) = @_;

    say "  Rolling back workflow...";

    for (my $i = $failed_step - 1; $i >= 0; $i--) {
        my $step = $self->steps->[$i];

        if ($step->{rollback}) {
            say "    Rolling back: $step->{name}";
            eval { $step->{rollback}->($self->context) };
            warn "    Rollback failed: $@" if $@;
        }
    }
}

# Example workflow: Deploy application
package main;

my $workflow = Workflow->new(name => 'Deploy Application');

$workflow->add_step(
    'Backup current version',
    sub {
        my $ctx = shift;
        system("cp -r /app /app.backup.$ctx->{timestamp}") == 0
            or die "Backup failed";
        $ctx->{backup_dir} = "/app.backup.$ctx->{timestamp}";
    },
    rollback => sub {
        my $ctx = shift;
        system("rm -rf $ctx->{backup_dir}");
    },
);

$workflow->add_step(
    'Download new version',
    sub {
        my $ctx = shift;
        system("wget -O /tmp/app.tar.gz $ctx->{download_url}") == 0
            or die "Download failed";
        $ctx->{archive} = '/tmp/app.tar.gz';
    },
    retry => 3,
    timeout => 300,
    rollback => sub {
        my $ctx = shift;
        unlink $ctx->{archive} if $ctx->{archive};
    },
);

$workflow->add_step(
    'Extract and install',
    sub {
        my $ctx = shift;
        system("tar -xzf $ctx->{archive} -C /app") == 0
            or die "Extraction failed";
    },
    rollback => sub {
        my $ctx = shift;
        system("rm -rf /app/*");
        system("cp -r $ctx->{backup_dir}/* /app/");
    },
);

$workflow->add_step(
    'Run database migrations',
    sub {
        my $ctx = shift;
        system("/app/bin/migrate") == 0
            or die "Migrations failed";
    },
);

$workflow->add_step(
    'Restart services',
    sub {
        my $ctx = shift;
        system("systemctl restart app.service") == 0
            or die "Service restart failed";
    },
);

$workflow->add_step(
    'Health check',
    sub {
        my $ctx = shift;
        sleep(5);  # Give service time to start

        my $response = `curl -s -o /dev/null -w "%{http_code}" http://localhost/health`;
        die "Health check failed: HTTP $response" unless $response eq '200';
    },
    retry => 3,
);

# Set up workflow handlers
$workflow->on_error(sub {
    my $wf = shift;
    system("echo 'Deployment failed at step: " .
           $wf->steps->[$wf->current_step]{name} .
           "' | mail -s 'Deployment Failed' ops@example.com");
});

$workflow->on_success(sub {
    system("echo 'Deployment completed successfully' | " .
           "mail -s 'Deployment Success' ops@example.com");
});

# Execute workflow
$workflow->context({
    timestamp => time(),
    download_url => 'https://releases.example.com/app-v2.0.tar.gz',
});

$workflow->execute();
```

## Task Scheduling and Queuing

### Advanced Task Scheduler

```perl
#!/usr/bin/env perl
package TaskScheduler;
use Modern::Perl '2023';
use Moo;
use Schedule::Cron;
use Parallel::ForkManager;
use Time::HiRes qw(time sleep);
use POSIX qw(strftime);

has 'cron' => (is => 'lazy');
has 'max_workers' => (is => 'ro', default => 5);
has 'tasks' => (is => 'ro', default => sub { [] });
has 'running_tasks' => (is => 'rw', default => sub { {} });

sub _build_cron {
    my $self = shift;
    return Schedule::Cron->new($self);
}

sub add_task {
    my ($self, %task) = @_;

    push @{$self->tasks}, {
        name => $task{name},
        schedule => $task{schedule},
        handler => $task{handler},
        timeout => $task{timeout} // 3600,
        max_runtime => $task{max_runtime},
        exclusive => $task{exclusive} // 0,
        enabled => 1,
    };

    # Add to cron scheduler
    $self->cron->add_entry(
        $task{schedule},
        sub { $self->run_task($task{name}) }
    );

    return $self;
}

sub run_task {
    my ($self, $task_name) = @_;

    my ($task) = grep { $_->{name} eq $task_name } @{$self->tasks};
    return unless $task && $task->{enabled};

    # Check exclusivity
    if ($task->{exclusive} && exists $self->running_tasks->{$task_name}) {
        warn "Task $task_name is already running (exclusive)";
        return;
    }

    # Fork to run task
    my $pid = fork();
    die "Fork failed: $!" unless defined $pid;

    if ($pid == 0) {
        # Child process
        $0 = "task: $task_name";

        # Set up timeout
        if ($task->{timeout}) {
            $SIG{ALRM} = sub { die "Task timeout\n" };
            alarm($task->{timeout});
        }

        # Run task
        eval {
            $task->{handler}->();
        };

        if ($@) {
            $self->log_error($task_name, $@);
        }

        exit(0);
    } else {
        # Parent process
        $self->running_tasks->{$task_name} = {
            pid => $pid,
            started => time(),
        };

        # Clean up finished tasks
        $self->reap_tasks();
    }
}

sub reap_tasks {
    my $self = shift;

    for my $task_name (keys %{$self->running_tasks}) {
        my $task_info = $self->running_tasks->{$task_name};
        my $pid = $task_info->{pid};

        my $kid = waitpid($pid, WNOHANG);
        if ($kid == $pid) {
            delete $self->running_tasks->{$task_name};
            $self->log("Task $task_name completed (PID: $pid)");
        }
    }
}

sub start {
    my $self = shift;

    $self->log("Scheduler starting with " . scalar(@{$self->tasks}) . " tasks");

    # Set up signal handlers
    $SIG{CHLD} = sub { $self->reap_tasks() };
    $SIG{TERM} = $SIG{INT} = sub {
        $self->log("Scheduler shutting down...");
        $self->stop();
        exit(0);
    };

    # Run scheduler
    $self->cron->run();
}

sub stop {
    my $self = shift;

    # Kill running tasks
    for my $task_name (keys %{$self->running_tasks}) {
        my $pid = $self->running_tasks->{$task_name}{pid};
        kill 'TERM', $pid;
    }

    # Wait for tasks to finish
    my $timeout = 10;
    while (keys %{$self->running_tasks} && $timeout-- > 0) {
        sleep(1);
        $self->reap_tasks();
    }

    # Force kill if necessary
    for my $task_name (keys %{$self->running_tasks}) {
        my $pid = $self->running_tasks->{$task_name}{pid};
        kill 'KILL', $pid;
    }
}

sub log {
    my ($self, $message) = @_;
    my $timestamp = strftime("%Y-%m-%d %H:%M:%S", localtime);
    say "[$timestamp] $message";
}

sub log_error {
    my ($self, $task, $error) = @_;
    $self->log("ERROR in $task: $error");
    # Could also send alerts, write to error log, etc.
}

# Example usage
package main;

my $scheduler = TaskScheduler->new(max_workers => 3);

# Add scheduled tasks
$scheduler->add_task(
    name => 'cleanup_temp',
    schedule => '0 * * * *',  # Every hour
    handler => sub {
        system('find /tmp -type f -mtime +1 -delete');
    },
);

$scheduler->add_task(
    name => 'backup_database',
    schedule => '0 2 * * *',  # Daily at 2 AM
    handler => sub {
        my $date = strftime("%Y%m%d", localtime);
        system("mysqldump mydb | gzip > /backup/mydb_$date.sql.gz");
    },
    exclusive => 1,
    timeout => 3600,
);

$scheduler->add_task(
    name => 'sync_files',
    schedule => '*/15 * * * *',  # Every 15 minutes
    handler => sub {
        system('rsync -av /source/ /destination/');
    },
);

$scheduler->add_task(
    name => 'generate_reports',
    schedule => '0 6 * * 1',  # Weekly on Monday at 6 AM
    handler => sub {
        system('/usr/local/bin/generate_weekly_reports.pl');
    },
    timeout => 7200,
);

# Start the scheduler
$scheduler->start();
```

## Complex Automation Pipelines

### Data Processing Pipeline

```perl
#!/usr/bin/env perl
package DataPipeline;
use Modern::Perl '2023';
use Moo;
use Parallel::ForkManager;
use File::Temp;
use JSON::XS;

has 'stages' => (is => 'ro', default => sub { [] });
has 'max_parallel' => (is => 'ro', default => 4);
has 'checkpoint_dir' => (is => 'ro', default => '/var/lib/pipeline');

sub add_stage {
    my ($self, $name, $processor, %options) = @_;

    push @{$self->stages}, {
        name => $name,
        processor => $processor,
        parallel => $options{parallel} // 1,
        batch_size => $options{batch_size} // 100,
        checkpoint => $options{checkpoint} // 1,
    };

    return $self;
}

sub run {
    my ($self, $input_data) = @_;

    my $data = $input_data;

    for my $stage (@{$self->stages}) {
        say "Running stage: $stage->{name}";

        # Load checkpoint if exists
        if ($stage->{checkpoint}) {
            my $checkpoint_data = $self->load_checkpoint($stage->{name});
            if ($checkpoint_data) {
                say "  Resuming from checkpoint";
                $data = $checkpoint_data;
                next;
            }
        }

        # Process stage
        $data = $self->process_stage($stage, $data);

        # Save checkpoint
        if ($stage->{checkpoint}) {
            $self->save_checkpoint($stage->{name}, $data);
        }
    }

    return $data;
}

sub process_stage {
    my ($self, $stage, $input_data) = @_;

    if ($stage->{parallel} > 1 && ref $input_data eq 'ARRAY') {
        return $self->process_parallel($stage, $input_data);
    } else {
        return $stage->{processor}->($input_data);
    }
}

sub process_parallel {
    my ($self, $stage, $data) = @_;

    my $pm = Parallel::ForkManager->new($stage->{parallel});
    my @results;

    # Set up result collection
    $pm->run_on_finish(sub {
        my ($pid, $exit_code, $ident, $exit_signal, $core_dump, $result) = @_;
        push @results, @$result if $result;
    });

    # Process in batches
    my $batch_size = $stage->{batch_size};
    for (my $i = 0; $i < @$data; $i += $batch_size) {
        my $end = $i + $batch_size - 1;
        $end = $#$data if $end > $#$data;

        my @batch = @$data[$i..$end];

        $pm->start and next;

        # Child process
        my $batch_results = $stage->{processor}->(\@batch);
        $pm->finish(0, $batch_results);
    }

    $pm->wait_all_children;

    return \@results;
}

sub save_checkpoint {
    my ($self, $stage_name, $data) = @_;

    mkdir $self->checkpoint_dir unless -d $self->checkpoint_dir;

    my $file = "$self->{checkpoint_dir}/$stage_name.checkpoint";
    open my $fh, '>', $file or die "Can't save checkpoint: $!";
    print $fh encode_json($data);
    close $fh;
}

sub load_checkpoint {
    my ($self, $stage_name) = @_;

    my $file = "$self->{checkpoint_dir}/$stage_name.checkpoint";
    return unless -f $file;

    open my $fh, '<', $file or die "Can't load checkpoint: $!";
    local $/;
    my $json = <$fh>;
    close $fh;

    return decode_json($json);
}

sub clear_checkpoints {
    my $self = shift;
    system("rm -f $self->{checkpoint_dir}/*.checkpoint");
}

# Example: ETL Pipeline
package main;

my $pipeline = DataPipeline->new();

# Stage 1: Extract
$pipeline->add_stage(
    'extract',
    sub {
        my $sources = shift;
        my @data;

        for my $source (@$sources) {
            say "  Extracting from $source";
            # Simulate data extraction
            push @data, map { { source => $source, id => $_, value => rand() } } 1..1000;
        }

        return \@data;
    },
);

# Stage 2: Transform
$pipeline->add_stage(
    'transform',
    sub {
        my $records = shift;

        for my $record (@$records) {
            # Normalize values
            $record->{normalized} = $record->{value} * 100;

            # Add metadata
            $record->{processed_at} = time();

            # Categorize
            $record->{category} = $record->{normalized} > 50 ? 'high' : 'low';
        }

        return $records;
    },
    parallel => 4,
    batch_size => 250,
);

# Stage 3: Validate
$pipeline->add_stage(
    'validate',
    sub {
        my $records = shift;
        my @valid;

        for my $record (@$records) {
            # Validation rules
            next unless $record->{id};
            next unless defined $record->{normalized};
            next if $record->{normalized} < 0 || $record->{normalized} > 100;

            push @valid, $record;
        }

        say "  Validated " . scalar(@valid) . " of " . scalar(@$records) . " records";
        return \@valid;
    },
);

# Stage 4: Load
$pipeline->add_stage(
    'load',
    sub {
        my $records = shift;

        # Group by category
        my %by_category;
        for my $record (@$records) {
            push @{$by_category{$record->{category}}}, $record;
        }

        # Save to different destinations
        for my $category (keys %by_category) {
            my $file = "/data/output_$category.json";
            open my $fh, '>', $file or die $!;
            print $fh encode_json($by_category{$category});
            close $fh;
            say "  Saved " . scalar(@{$by_category{$category}}) . " records to $file";
        }

        return { status => 'completed', record_count => scalar(@$records) };
    },
);

# Run pipeline
my $sources = ['/data/source1', '/data/source2', '/data/source3'];
my $result = $pipeline->run($sources);

say "Pipeline completed: " . encode_json($result);
```

## Deployment Automation

### Automated Deployment System

```perl
#!/usr/bin/env perl
package DeploymentAutomation;
use Modern::Perl '2023';
use YAML::XS qw(LoadFile);
use Net::SSH::Perl;
use Git::Repository;

sub new {
    my ($class, $config_file) = @_;

    my $self = {
        config => LoadFile($config_file),
        errors => [],
    };

    return bless $self, $class;
}

sub deploy {
    my ($self, $environment, $version) = @_;

    say "Starting deployment to $environment (version: $version)";

    # Pre-deployment checks
    return unless $self->pre_deployment_checks($environment);

    # Get deployment plan
    my $plan = $self->create_deployment_plan($environment, $version);

    # Execute deployment
    for my $step (@{$plan->{steps}}) {
        unless ($self->execute_step($step, $environment)) {
            $self->rollback($environment, $plan);
            return 0;
        }
    }

    # Post-deployment validation
    unless ($self->validate_deployment($environment)) {
        $self->rollback($environment, $plan);
        return 0;
    }

    say "Deployment completed successfully";
    $self->notify_success($environment, $version);

    return 1;
}

sub pre_deployment_checks {
    my ($self, $environment) = @_;

    say "Running pre-deployment checks...";

    # Check if environment is locked
    if ($self->is_environment_locked($environment)) {
        push @{$self->errors}, "Environment $environment is locked";
        return 0;
    }

    # Check dependencies
    unless ($self->check_dependencies($environment)) {
        push @{$self->errors}, "Dependency check failed";
        return 0;
    }

    # Check disk space
    for my $server (@{$self->config->{environments}{$environment}{servers}}) {
        my $space = $self->check_disk_space($server);
        if ($space < 1024 * 1024 * 1024) {  # 1GB minimum
            push @{$self->errors}, "Insufficient disk space on $server";
            return 0;
        }
    }

    say "  All checks passed";
    return 1;
}

sub create_deployment_plan {
    my ($self, $environment, $version) = @_;

    my $plan = {
        environment => $environment,
        version => $version,
        timestamp => time(),
        steps => [],
    };

    # Build deployment steps
    push @{$plan->{steps}}, {
        name => 'Lock environment',
        action => sub { $self->lock_environment($environment) },
        rollback => sub { $self->unlock_environment($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Backup current version',
        action => sub { $self->backup_current($environment) },
        rollback => sub { $self->cleanup_backup($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Download new version',
        action => sub { $self->download_version($version) },
        rollback => sub { $self->cleanup_download($version) },
    };

    push @{$plan->{steps}}, {
        name => 'Stop services',
        action => sub { $self->stop_services($environment) },
        rollback => sub { $self->start_services($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Deploy code',
        action => sub { $self->deploy_code($environment, $version) },
        rollback => sub { $self->restore_backup($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Run migrations',
        action => sub { $self->run_migrations($environment) },
        rollback => sub { $self->rollback_migrations($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Start services',
        action => sub { $self->start_services($environment) },
    };

    push @{$plan->{steps}}, {
        name => 'Warm up cache',
        action => sub { $self->warm_cache($environment) },
    };

    return $plan;
}

sub execute_step {
    my ($self, $step, $environment) = @_;

    say "  Executing: $step->{name}";

    eval {
        $step->{action}->();
    };

    if ($@) {
        push @{$self->errors}, "Step '$step->{name}' failed: $@";
        return 0;
    }

    return 1;
}

sub deploy_code {
    my ($self, $environment, $version) = @_;

    my $servers = $self->config->{environments}{$environment}{servers};

    for my $server (@$servers) {
        say "    Deploying to $server";

        my $ssh = Net::SSH::Perl->new($server);
        $ssh->login($self->config->{ssh_user}, $self->config->{ssh_key});

        # Copy files
        my $deploy_path = $self->config->{deploy_path};
        my ($stdout, $stderr, $exit) = $ssh->cmd(
            "rm -rf $deploy_path.new && " .
            "cp -r $deploy_path.download $deploy_path.new && " .
            "mv $deploy_path $deploy_path.old && " .
            "mv $deploy_path.new $deploy_path"
        );

        die "Deployment failed on $server: $stderr" if $exit != 0;
    }
}

sub validate_deployment {
    my ($self, $environment) = @_;

    say "Validating deployment...";

    my $servers = $self->config->{environments}{$environment}{servers};
    my $health_endpoint = $self->config->{health_check_endpoint};

    for my $server (@$servers) {
        my $url = "http://$server$health_endpoint";
        my $response = `curl -s -o /dev/null -w "%{http_code}" $url`;

        unless ($response eq '200') {
            push @{$self->errors}, "Health check failed on $server (HTTP $response)";
            return 0;
        }
    }

    say "  Validation passed";
    return 1;
}

sub rollback {
    my ($self, $environment, $plan) = @_;

    say "ROLLING BACK deployment...";

    # Execute rollback actions in reverse order
    for my $step (reverse @{$plan->{steps}}) {
        next unless $step->{rollback};

        say "  Rolling back: $step->{name}";
        eval { $step->{rollback}->() };
        warn "    Rollback failed: $@" if $@;
    }

    $self->notify_rollback($environment, $plan->{version});
}

sub notify_success {
    my ($self, $environment, $version) = @_;

    my $message = <<END;
Deployment Successful!

Environment: $environment
Version: $version
Time: @{[scalar localtime]}

The deployment completed successfully and all health checks passed.
END

    $self->send_notification('Deployment Success', $message);
}

sub notify_rollback {
    my ($self, $environment, $version) = @_;

    my $message = <<END;
Deployment Failed - Rollback Executed

Environment: $environment
Version: $version
Time: @{[scalar localtime]}

Errors:
@{[join("\n", map { "  - $_" } @{$self->errors})]}

The deployment has been rolled back to the previous version.
END

    $self->send_notification('Deployment Rollback', $message);
}

sub send_notification {
    my ($self, $subject, $message) = @_;

    # Send email
    system("echo '$message' | mail -s '$subject' " .
           $self->config->{notification_email});

    # Send to Slack
    if ($self->config->{slack_webhook}) {
        my $json = encode_json({ text => "$subject\n$message" });
        system("curl -X POST -H 'Content-type: application/json' " .
               "--data '$json' " . $self->config->{slack_webhook});
    }
}

# Stub implementations
sub is_environment_locked { 0 }
sub lock_environment { 1 }
sub unlock_environment { 1 }
sub check_dependencies { 1 }
sub check_disk_space { 2 * 1024 * 1024 * 1024 }  # 2GB
sub backup_current { 1 }
sub cleanup_backup { 1 }
sub download_version { 1 }
sub cleanup_download { 1 }
sub stop_services { 1 }
sub start_services { 1 }
sub restore_backup { 1 }
sub run_migrations { 1 }
sub rollback_migrations { 1 }
sub warm_cache { 1 }

# Usage
package main;

my $deployment = DeploymentAutomation->new('deployment.yaml');
$deployment->deploy('production', 'v2.3.1');
```

## Best Practices

1. **Always use locking** - Prevent concurrent execution
2. **Log everything** - Crucial for debugging cron jobs
3. **Handle failures gracefully** - Plan for rollbacks
4. **Set appropriate timeouts** - Prevent hung jobs
5. **Monitor your automation** - Know when things fail
6. **Use configuration files** - Don't hardcode values
7. **Test in staging first** - Never test in production
8. **Document dependencies** - Make deployment reproducible
9. **Version your automation scripts** - Track changes
10. **Implement health checks** - Verify successful completion

## Conclusion

Automation is about more than just scheduling scripts—it's about building reliable, maintainable systems that reduce toil and increase consistency. Perl's combination of system integration, text processing, and CPAN modules makes it ideal for building sophisticated automation workflows.

Remember: good automation is invisible when it works and informative when it doesn't. Build systems that fail gracefully, recover automatically when possible, and always keep humans in the loop for critical decisions.

---

*Next: RESTful APIs and Web Services. We'll explore how to build and consume web services with Perl.*