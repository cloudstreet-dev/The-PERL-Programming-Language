# Chapter 12: Database Operations with DBI

> "A database is just a spreadsheet on steroids, and Perl is the personal trainer." - Anonymous DBA

Databases are where your data lives, and Perl's DBI (Database Independent Interface) is how you talk to them. Whether it's MySQL, PostgreSQL, SQLite, or Oracle, DBI provides a consistent interface that makes database programming in Perl both powerful and portable. This chapter covers everything from basic queries to advanced patterns like connection pooling and migrations.

## DBI Fundamentals

### Connecting to Databases

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use DBI;

# Basic connection
sub connect_to_database {
    my ($database, $user, $password) = @_;

    # Different database drivers
    my %dsn = (
        mysql  => "DBI:mysql:database=$database;host=localhost;port=3306",
        pg     => "DBI:Pg:dbname=$database;host=localhost;port=5432",
        sqlite => "DBI:SQLite:dbname=$database.db",
        oracle => "DBI:Oracle:host=localhost;sid=$database;port=1521",
    );

    my $dbh = DBI->connect(
        $dsn{mysql},  # Or pg, sqlite, etc.
        $user,
        $password,
        {
            RaiseError => 1,        # Die on errors
            PrintError => 0,        # Don't warn on errors
            AutoCommit => 1,        # Commit automatically
            mysql_enable_utf8 => 1, # UTF-8 for MySQL
            pg_enable_utf8 => 1,    # UTF-8 for PostgreSQL
        }
    ) or die "Connection failed: $DBI::errstr";

    return $dbh;
}

# Connection with error handling
sub safe_connect {
    my ($dsn, $user, $pass) = @_;

    my $dbh;
    my $attempts = 0;
    my $max_attempts = 3;

    while ($attempts < $max_attempts) {
        $attempts++;

        eval {
            $dbh = DBI->connect($dsn, $user, $pass, {
                RaiseError => 1,
                PrintError => 0,
                AutoCommit => 1,
            });
        };

        if ($@) {
            warn "Connection attempt $attempts failed: $@";
            sleep(2 ** $attempts);  # Exponential backoff
        } else {
            last;  # Success
        }
    }

    die "Failed to connect after $max_attempts attempts" unless $dbh;

    return $dbh;
}

# SQLite in-memory database for testing
sub create_test_db {
    my $dbh = DBI->connect("DBI:SQLite:dbname=:memory:", "", "", {
        RaiseError => 1,
        PrintError => 0,
    });

    # Create test schema
    $dbh->do(<<'SQL');
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
SQL

    return $dbh;
}
```

### Basic CRUD Operations

```perl
# CREATE - Insert data
sub insert_user {
    my ($dbh, $username, $email) = @_;

    my $sql = "INSERT INTO users (username, email) VALUES (?, ?)";
    my $sth = $dbh->prepare($sql);
    $sth->execute($username, $email);

    # Get auto-increment ID
    my $user_id = $dbh->last_insert_id(undef, undef, 'users', 'id');

    return $user_id;
}

# READ - Select data
sub get_user {
    my ($dbh, $user_id) = @_;

    my $sql = "SELECT * FROM users WHERE id = ?";
    my $sth = $dbh->prepare($sql);
    $sth->execute($user_id);

    my $user = $sth->fetchrow_hashref();
    return $user;
}

sub get_all_users {
    my ($dbh) = @_;

    my $sql = "SELECT * FROM users ORDER BY created_at DESC";
    my $sth = $dbh->prepare($sql);
    $sth->execute();

    my @users;
    while (my $row = $sth->fetchrow_hashref()) {
        push @users, $row;
    }

    return \@users;
}

# UPDATE - Modify data
sub update_user {
    my ($dbh, $user_id, $updates) = @_;

    my @fields = keys %$updates;
    my @values = values %$updates;
    push @values, $user_id;

    my $sql = "UPDATE users SET " .
              join(", ", map { "$_ = ?" } @fields) .
              " WHERE id = ?";

    my $sth = $dbh->prepare($sql);
    my $rows = $sth->execute(@values);

    return $rows;  # Number of rows affected
}

# DELETE - Remove data
sub delete_user {
    my ($dbh, $user_id) = @_;

    my $sql = "DELETE FROM users WHERE id = ?";
    my $sth = $dbh->prepare($sql);
    my $rows = $sth->execute($user_id);

    return $rows;
}
```

### Prepared Statements and Placeholders

```perl
# Safe queries with placeholders
sub search_users {
    my ($dbh, $search_term) = @_;

    # NEVER do this - SQL injection vulnerability!
    # my $sql = "SELECT * FROM users WHERE username LIKE '%$search_term%'";

    # DO this instead - safe with placeholders
    my $sql = "SELECT * FROM users WHERE username LIKE ?";
    my $sth = $dbh->prepare($sql);
    $sth->execute("%$search_term%");

    return $sth->fetchall_arrayref({});
}

# Cached prepared statements
sub get_statement_handle {
    my ($dbh, $sql) = @_;

    state %cache;
    $cache{$sql} //= $dbh->prepare_cached($sql);

    return $cache{$sql};
}

# Using cached statements
sub fast_user_lookup {
    my ($dbh, @user_ids) = @_;

    my $sth = get_statement_handle($dbh,
        "SELECT * FROM users WHERE id = ?"
    );

    my @users;
    for my $id (@user_ids) {
        $sth->execute($id);
        push @users, $sth->fetchrow_hashref();
    }

    return \@users;
}
```

## Transactions

### Basic Transaction Handling

```perl
sub transfer_funds {
    my ($dbh, $from_account, $to_account, $amount) = @_;

    # Start transaction
    $dbh->begin_work;

    eval {
        # Check balance
        my $balance = $dbh->selectrow_array(
            "SELECT balance FROM accounts WHERE id = ?",
            undef, $from_account
        );

        die "Insufficient funds" if $balance < $amount;

        # Debit from account
        $dbh->do(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            undef, $amount, $from_account
        );

        # Credit to account
        $dbh->do(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            undef, $amount, $to_account
        );

        # Log transaction
        $dbh->do(
            "INSERT INTO transactions (from_id, to_id, amount) VALUES (?, ?, ?)",
            undef, $from_account, $to_account, $amount
        );

        # Commit if all successful
        $dbh->commit;
    };

    if ($@) {
        # Rollback on error
        $dbh->rollback;
        die "Transaction failed: $@";
    }

    return 1;
}

# Transaction wrapper
sub with_transaction {
    my ($dbh, $code) = @_;

    my $old_autocommit = $dbh->{AutoCommit};
    $dbh->{AutoCommit} = 0;

    my $result;
    eval {
        $result = $code->();
        $dbh->commit;
    };

    if ($@) {
        my $error = $@;
        eval { $dbh->rollback };
        $dbh->{AutoCommit} = $old_autocommit;
        die $error;
    }

    $dbh->{AutoCommit} = $old_autocommit;
    return $result;
}

# Using the wrapper
with_transaction($dbh, sub {
    insert_user($dbh, 'alice', 'alice@example.com');
    insert_user($dbh, 'bob', 'bob@example.com');
    # All or nothing
});
```

## Advanced DBI Features

### Batch Operations

```perl
# Bulk insert
sub bulk_insert_users {
    my ($dbh, $users) = @_;

    my $sql = "INSERT INTO users (username, email) VALUES (?, ?)";
    my $sth = $dbh->prepare($sql);

    $dbh->begin_work;

    eval {
        for my $user (@$users) {
            $sth->execute($user->{username}, $user->{email});
        }
        $dbh->commit;
    };

    if ($@) {
        $dbh->rollback;
        die "Bulk insert failed: $@";
    }

    return scalar(@$users);
}

# Using execute_array for better performance
sub optimized_bulk_insert {
    my ($dbh, $users) = @_;

    my $sql = "INSERT INTO users (username, email) VALUES (?, ?)";
    my $sth = $dbh->prepare($sql);

    my @usernames = map { $_->{username} } @$users;
    my @emails = map { $_->{email} } @$users;

    my @tuple_status;
    my $rows = $sth->execute_array(
        { ArrayTupleStatus => \@tuple_status },
        \@usernames,
        \@emails
    );

    # Check for errors
    for my $i (0..$#tuple_status) {
        if (ref $tuple_status[$i]) {
            warn "Row $i failed: @{$tuple_status[$i]}";
        }
    }

    return $rows;
}
```

### Database Metadata

```perl
# Get table information
sub describe_table {
    my ($dbh, $table) = @_;

    my $sth = $dbh->column_info(undef, undef, $table, '%');
    my $columns = $sth->fetchall_arrayref({});

    my @info;
    for my $col (@$columns) {
        push @info, {
            name => $col->{COLUMN_NAME},
            type => $col->{TYPE_NAME},
            size => $col->{COLUMN_SIZE},
            nullable => $col->{NULLABLE} ? 'YES' : 'NO',
            default => $col->{COLUMN_DEF},
        };
    }

    return \@info;
}

# List all tables
sub list_tables {
    my ($dbh) = @_;

    my @tables = $dbh->tables(undef, undef, undef, 'TABLE');

    # Clean up table names (remove quotes/schema)
    @tables = map { s/^.*\.//; s/["'`]//g; $_ } @tables;

    return \@tables;
}

# Get foreign keys
sub get_foreign_keys {
    my ($dbh, $table) = @_;

    my $sth = $dbh->foreign_key_info(
        undef, undef, undef,  # Primary table
        undef, undef, $table  # Foreign table
    );

    return [] unless $sth;

    my @foreign_keys;
    while (my $row = $sth->fetchrow_hashref()) {
        push @foreign_keys, {
            constraint => $row->{FK_NAME},
            column => $row->{FK_COLUMN_NAME},
            ref_table => $row->{PK_TABLE_NAME},
            ref_column => $row->{PK_COLUMN_NAME},
        };
    }

    return \@foreign_keys;
}
```

## Database Abstraction Layer

### Building a Simple ORM

```perl
package SimpleORM;
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

sub new($class, $dbh, $table) {
    return bless {
        dbh => $dbh,
        table => $table,
    }, $class;
}

sub find($self, $id) {
    my $sql = "SELECT * FROM $self->{table} WHERE id = ?";
    my $row = $self->{dbh}->selectrow_hashref($sql, undef, $id);
    return $row;
}

sub find_by($self, $conditions) {
    my @where = map { "$_ = ?" } keys %$conditions;
    my @values = values %$conditions;

    my $sql = "SELECT * FROM $self->{table} WHERE " . join(" AND ", @where);
    my $sth = $self->{dbh}->prepare($sql);
    $sth->execute(@values);

    return $sth->fetchall_arrayref({});
}

sub create($self, $data) {
    my @fields = keys %$data;
    my @values = values %$data;
    my @placeholders = ('?') x @fields;

    my $sql = sprintf(
        "INSERT INTO %s (%s) VALUES (%s)",
        $self->{table},
        join(", ", @fields),
        join(", ", @placeholders)
    );

    my $sth = $self->{dbh}->prepare($sql);
    $sth->execute(@values);

    return $self->{dbh}->last_insert_id(undef, undef, $self->{table}, 'id');
}

sub update($self, $id, $data) {
    my @fields = keys %$data;
    my @values = values %$data;
    push @values, $id;

    my $sql = sprintf(
        "UPDATE %s SET %s WHERE id = ?",
        $self->{table},
        join(", ", map { "$_ = ?" } @fields)
    );

    my $sth = $self->{dbh}->prepare($sql);
    return $sth->execute(@values);
}

sub delete($self, $id) {
    my $sql = "DELETE FROM $self->{table} WHERE id = ?";
    my $sth = $self->{dbh}->prepare($sql);
    return $sth->execute($id);
}

sub count($self, $conditions = {}) {
    my $sql = "SELECT COUNT(*) FROM $self->{table}";

    if (%$conditions) {
        my @where = map { "$_ = ?" } keys %$conditions;
        $sql .= " WHERE " . join(" AND ", @where);
    }

    my @values = values %$conditions;
    my ($count) = $self->{dbh}->selectrow_array($sql, undef, @values);

    return $count;
}

# Usage
package main;

my $users = SimpleORM->new($dbh, 'users');

# Create
my $id = $users->create({
    username => 'charlie',
    email => 'charlie@example.com',
});

# Read
my $user = $users->find($id);
my $active_users = $users->find_by({ status => 'active' });

# Update
$users->update($id, { email => 'newemail@example.com' });

# Delete
$users->delete($id);

# Count
my $total = $users->count();
my $active = $users->count({ status => 'active' });
```

## Connection Pooling

### Simple Connection Pool

```perl
package ConnectionPool;
use Modern::Perl '2023';
use Time::HiRes qw(time);

sub new {
    my ($class, $config) = @_;

    return bless {
        dsn => $config->{dsn},
        user => $config->{user},
        password => $config->{password},
        min_connections => $config->{min_connections} // 2,
        max_connections => $config->{max_connections} // 10,
        idle_timeout => $config->{idle_timeout} // 300,
        connections => [],
        in_use => {},
    }, $class;
}

sub get_connection {
    my ($self) = @_;

    # Return idle connection if available
    for my $conn (@{$self->{connections}}) {
        next if $self->{in_use}{$conn};

        # Check if connection is still alive
        if ($self->ping_connection($conn)) {
            $self->{in_use}{$conn} = time();
            return $conn;
        } else {
            # Remove dead connection
            $self->remove_connection($conn);
        }
    }

    # Create new connection if under limit
    if (@{$self->{connections}} < $self->{max_connections}) {
        my $conn = $self->create_connection();
        push @{$self->{connections}}, $conn;
        $self->{in_use}{$conn} = time();
        return $conn;
    }

    # Wait for available connection
    die "Connection pool exhausted";
}

sub release_connection {
    my ($self, $conn) = @_;

    delete $self->{in_use}{$conn};

    # Remove old idle connections
    $self->cleanup_idle_connections();
}

sub create_connection {
    my ($self) = @_;

    return DBI->connect(
        $self->{dsn},
        $self->{user},
        $self->{password},
        {
            RaiseError => 1,
            PrintError => 0,
            AutoCommit => 1,
        }
    );
}

sub ping_connection {
    my ($self, $conn) = @_;

    return eval { $conn->ping() };
}

sub remove_connection {
    my ($self, $conn) = @_;

    @{$self->{connections}} = grep { $_ != $conn } @{$self->{connections}};
    delete $self->{in_use}{$conn};

    eval { $conn->disconnect() };
}

sub cleanup_idle_connections {
    my ($self) = @_;

    my $now = time();
    my $timeout = $self->{idle_timeout};

    for my $conn (@{$self->{connections}}) {
        next if $self->{in_use}{$conn};

        my $last_used = $self->{last_used}{$conn} // 0;
        if ($now - $last_used > $timeout) {
            $self->remove_connection($conn);
        }
    }

    # Maintain minimum connections
    while (@{$self->{connections}} < $self->{min_connections}) {
        push @{$self->{connections}}, $self->create_connection();
    }
}

sub shutdown {
    my ($self) = @_;

    for my $conn (@{$self->{connections}}) {
        eval { $conn->disconnect() };
    }

    $self->{connections} = [];
    $self->{in_use} = {};
}

# Usage
my $pool = ConnectionPool->new({
    dsn => 'DBI:mysql:database=myapp',
    user => 'dbuser',
    password => 'dbpass',
    min_connections => 2,
    max_connections => 10,
});

my $dbh = $pool->get_connection();
# Use $dbh...
$pool->release_connection($dbh);
```

## Database Migrations

### Migration System

```perl
package MigrationRunner;
use Modern::Perl '2023';
use File::Basename;

sub new {
    my ($class, $dbh, $migration_dir) = @_;

    return bless {
        dbh => $dbh,
        migration_dir => $migration_dir,
    }, $class;
}

sub setup {
    my ($self) = @_;

    $self->{dbh}->do(<<'SQL');
        CREATE TABLE IF NOT EXISTS migrations (
            version INTEGER PRIMARY KEY,
            filename TEXT NOT NULL,
            applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
SQL
}

sub get_current_version {
    my ($self) = @_;

    my ($version) = $self->{dbh}->selectrow_array(
        "SELECT MAX(version) FROM migrations"
    );

    return $version // 0;
}

sub get_pending_migrations {
    my ($self) = @_;

    my $current_version = $self->get_current_version();
    my @migrations;

    opendir my $dh, $self->{migration_dir} or die $!;
    while (my $file = readdir $dh) {
        next unless $file =~ /^(\d+)_(.+)\.sql$/;
        my ($version, $name) = ($1, $2);

        if ($version > $current_version) {
            push @migrations, {
                version => $version,
                name => $name,
                filename => $file,
                path => "$self->{migration_dir}/$file",
            };
        }
    }
    closedir $dh;

    return [ sort { $a->{version} <=> $b->{version} } @migrations ];
}

sub run_migration {
    my ($self, $migration) = @_;

    say "Running migration $migration->{version}: $migration->{name}";

    # Read SQL file
    open my $fh, '<', $migration->{path} or die $!;
    local $/;
    my $sql = <$fh>;
    close $fh;

    # Execute migration in transaction
    $self->{dbh}->begin_work;

    eval {
        # Split on semicolons (simple approach)
        my @statements = split /;\s*$/m, $sql;

        for my $statement (@statements) {
            next if $statement =~ /^\s*$/;
            $self->{dbh}->do($statement);
        }

        # Record migration
        $self->{dbh}->do(
            "INSERT INTO migrations (version, filename) VALUES (?, ?)",
            undef,
            $migration->{version},
            $migration->{filename}
        );

        $self->{dbh}->commit;
        say "  ✓ Migration $migration->{version} completed";
    };

    if ($@) {
        $self->{dbh}->rollback;
        die "Migration $migration->{version} failed: $@";
    }
}

sub migrate {
    my ($self) = @_;

    $self->setup();

    my $pending = $self->get_pending_migrations();

    if (@$pending == 0) {
        say "Database is up to date";
        return;
    }

    say "Found " . scalar(@$pending) . " pending migrations";

    for my $migration (@$pending) {
        $self->run_migration($migration);
    }

    say "All migrations completed successfully";
}

# Usage
my $migrator = MigrationRunner->new($dbh, './migrations');
$migrator->migrate();
```

## Real-World Example: Database-Backed Queue

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use DBI;
use JSON::XS;
use Time::HiRes qw(time sleep);

package DatabaseQueue;

sub new {
    my ($class, $dbh) = @_;

    my $self = bless { dbh => $dbh }, $class;
    $self->setup();

    return $self;
}

sub setup {
    my ($self) = @_;

    $self->{dbh}->do(<<'SQL');
        CREATE TABLE IF NOT EXISTS queue (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            queue_name TEXT NOT NULL,
            payload TEXT NOT NULL,
            status TEXT DEFAULT 'pending',
            attempts INTEGER DEFAULT 0,
            max_attempts INTEGER DEFAULT 3,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            locked_until TIMESTAMP,
            completed_at TIMESTAMP,
            error TEXT,
            INDEX idx_queue_status (queue_name, status),
            INDEX idx_locked_until (locked_until)
        )
SQL
}

sub enqueue {
    my ($self, $queue_name, $payload, $options) = @_;
    $options //= {};

    my $json = JSON::XS->new->utf8->canonical;
    my $payload_json = $json->encode($payload);

    my $id = $self->{dbh}->do(
        "INSERT INTO queue (queue_name, payload, max_attempts) VALUES (?, ?, ?)",
        undef,
        $queue_name,
        $payload_json,
        $options->{max_attempts} // 3
    );

    return $self->{dbh}->last_insert_id(undef, undef, 'queue', 'id');
}

sub dequeue {
    my ($self, $queue_name, $lock_duration) = @_;
    $lock_duration //= 60;  # 60 seconds default

    my $now = time();
    my $lock_until = $now + $lock_duration;

    # Start transaction
    $self->{dbh}->begin_work;

    my $job;
    eval {
        # Find next available job
        my $sql = <<'SQL';
            SELECT * FROM queue
            WHERE queue_name = ?
              AND status = 'pending'
              AND (locked_until IS NULL OR locked_until < ?)
            ORDER BY created_at ASC
            LIMIT 1
            FOR UPDATE
SQL

        $job = $self->{dbh}->selectrow_hashref($sql, undef, $queue_name, $now);

        if ($job) {
            # Lock the job
            $self->{dbh}->do(
                "UPDATE queue SET status = 'processing', locked_until = ?, attempts = attempts + 1 WHERE id = ?",
                undef,
                $lock_until,
                $job->{id}
            );

            # Decode payload
            my $json = JSON::XS->new->utf8;
            $job->{payload} = $json->decode($job->{payload});
        }

        $self->{dbh}->commit;
    };

    if ($@) {
        $self->{dbh}->rollback;
        die "Failed to dequeue: $@";
    }

    return $job;
}

sub complete {
    my ($self, $job_id) = @_;

    $self->{dbh}->do(
        "UPDATE queue SET status = 'completed', completed_at = CURRENT_TIMESTAMP WHERE id = ?",
        undef,
        $job_id
    );
}

sub fail {
    my ($self, $job_id, $error) = @_;

    my ($attempts, $max_attempts) = $self->{dbh}->selectrow_array(
        "SELECT attempts, max_attempts FROM queue WHERE id = ?",
        undef,
        $job_id
    );

    my $status = ($attempts >= $max_attempts) ? 'failed' : 'pending';

    $self->{dbh}->do(
        "UPDATE queue SET status = ?, error = ?, locked_until = NULL WHERE id = ?",
        undef,
        $status,
        $error,
        $job_id
    );
}

sub stats {
    my ($self, $queue_name) = @_;

    my $sql = <<'SQL';
        SELECT
            status,
            COUNT(*) as count
        FROM queue
        WHERE queue_name = ?
        GROUP BY status
SQL

    my $sth = $self->{dbh}->prepare($sql);
    $sth->execute($queue_name);

    my %stats;
    while (my $row = $sth->fetchrow_hashref()) {
        $stats{$row->{status}} = $row->{count};
    }

    return \%stats;
}

# Worker implementation
package Worker;

sub new {
    my ($class, $queue, $handler) = @_;

    return bless {
        queue => $queue,
        handler => $handler,
        running => 0,
    }, $class;
}

sub run {
    my ($self) = @_;

    $self->{running} = 1;
    $SIG{TERM} = $SIG{INT} = sub { $self->{running} = 0 };

    say "Worker started, processing queue...";

    while ($self->{running}) {
        my $job = $self->{queue}->dequeue('default', 60);

        if ($job) {
            $self->process_job($job);
        } else {
            sleep(1);  # Wait before checking again
        }
    }

    say "Worker stopped";
}

sub process_job {
    my ($self, $job) = @_;

    say "Processing job $job->{id}";

    eval {
        $self->{handler}->($job->{payload});
        $self->{queue}->complete($job->{id});
        say "Job $job->{id} completed";
    };

    if ($@) {
        my $error = $@;
        warn "Job $job->{id} failed: $error";
        $self->{queue}->fail($job->{id}, $error);
    }
}

# Usage
package main;

my $dbh = DBI->connect("DBI:SQLite:dbname=queue.db", "", "", {
    RaiseError => 1,
    PrintError => 0,
});

my $queue = DatabaseQueue->new($dbh);

# Producer
$queue->enqueue('default', { task => 'send_email', to => 'user@example.com' });
$queue->enqueue('default', { task => 'generate_report', id => 123 });

# Consumer
my $worker = Worker->new($queue, sub {
    my ($payload) = @_;

    if ($payload->{task} eq 'send_email') {
        say "Sending email to $payload->{to}";
        sleep(2);  # Simulate work
    } elsif ($payload->{task} eq 'generate_report') {
        say "Generating report $payload->{id}";
        sleep(3);  # Simulate work
    }
});

$worker->run();
```

## Best Practices

1. **Always use placeholders** - Never interpolate variables into SQL
2. **Use transactions for consistency** - Group related operations
3. **Handle connection failures** - Networks are unreliable
4. **Cache prepared statements** - Reuse for better performance
5. **Set appropriate timeouts** - Don't wait forever
6. **Use the right isolation level** - Balance consistency and performance
7. **Index your queries** - Check EXPLAIN plans
8. **Clean up resources** - Close statements and connections
9. **Log slow queries** - Monitor performance
10. **Test with production-like data** - Development data often differs

## Conclusion

DBI makes database programming in Perl consistent and powerful. Whether you're writing simple queries or building complex applications with migrations and connection pooling, DBI provides the tools you need. The key is understanding both DBI's features and your database's capabilities.

Remember: the database is often the bottleneck in applications. Write efficient queries, use appropriate indexes, and always measure performance with real data.

---

*Next: Configuration management and templating. We'll explore how to manage application configuration, generate dynamic content, and separate logic from presentation.*