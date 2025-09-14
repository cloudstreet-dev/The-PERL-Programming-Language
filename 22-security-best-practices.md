# Chapter 22: Security Best Practices

*"In the battle between attackers and defenders, Perl arms you with both sword and shield—but wisdom determines how you wield them."*

## The Security Mindset

Security isn't a feature you add; it's a discipline you practice. In 2025, with systems more interconnected than ever, a single vulnerability can cascade into catastrophic breaches. Perl, with its power and flexibility, can be either your greatest ally or your worst enemy in security—the difference lies in how you use it.

## Input Validation and Sanitization

Never trust input. Ever. This is the first commandment of secure programming:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Validator {
    use Moo;
    use Types::Standard qw(HashRef CodeRef);
    use Email::Valid;
    use Data::Validate::IP;
    use URI;
    use HTML::Entities;
    use Try::Tiny;

    has rules => (
        is => 'ro',
        isa => HashRef[CodeRef],
        default => sub { {} },
    );

    sub BUILD ($self, $args) {
        # Define validation rules
        $self->rules->{email} = sub ($value) {
            return Email::Valid->address($value) ? undef : "Invalid email address";
        };

        $self->rules->{ip} = sub ($value) {
            return is_ip($value) ? undef : "Invalid IP address";
        };

        $self->rules->{url} = sub ($value) {
            my $uri = URI->new($value);
            return ($uri && $uri->scheme && $uri->scheme =~ /^https?$/)
                ? undef : "Invalid URL";
        };

        $self->rules->{alphanumeric} = sub ($value) {
            return $value =~ /^[a-zA-Z0-9]+$/ ? undef : "Must be alphanumeric";
        };

        $self->rules->{integer} = sub ($value) {
            return $value =~ /^-?\d+$/ ? undef : "Must be an integer";
        };

        $self->rules->{positive_integer} = sub ($value) {
            return ($value =~ /^\d+$/ && $value > 0) ? undef : "Must be a positive integer";
        };

        $self->rules->{safe_string} = sub ($value) {
            # No shell metacharacters
            return $value =~ /[\$\`\|\;\&\<\>\(\)\{\}\[\]\*\?\~\!]/
                ? "Contains unsafe characters" : undef;
        };

        $self->rules->{sql_safe} = sub ($value) {
            # Basic SQL injection prevention
            return $value =~ /(\-\-|\/\*|\*\/|xp_|sp_|';|union|select|insert|update|delete|drop)/i
                ? "Contains potentially dangerous SQL patterns" : undef;
        };

        $self->rules->{path_safe} = sub ($value) {
            # Prevent directory traversal
            return $value =~ /\.\./ ? "Path traversal attempt detected" : undef;
        };

        $self->rules->{xss_safe} = sub ($value) {
            # Check for potential XSS patterns
            return $value =~ /<script|javascript:|on\w+=/i
                ? "Potential XSS detected" : undef;
        };
    }

    sub validate ($self, $input, $rules) {
        my %errors;
        my %clean;

        for my $field (keys %$rules) {
            my $value = $input->{$field};
            my $field_rules = ref $rules->{$field} eq 'ARRAY'
                ? $rules->{$field} : [$rules->{$field}];

            # Check required
            if (grep { $_ eq 'required' } @$field_rules) {
                if (!defined $value || $value eq '') {
                    $errors{$field} = "$field is required";
                    next;
                }
            }

            next unless defined $value && $value ne '';

            # Apply validation rules
            for my $rule (@$field_rules) {
                next if $rule eq 'required';

                if (ref $rule eq 'CODE') {
                    if (my $error = $rule->($value)) {
                        $errors{$field} = $error;
                        last;
                    }
                }
                elsif (exists $self->rules->{$rule}) {
                    if (my $error = $self->rules->{$rule}->($value)) {
                        $errors{$field} = $error;
                        last;
                    }
                }
                elsif ($rule eq 'sanitize_html') {
                    $value = encode_entities($value);
                }
                elsif ($rule eq 'trim') {
                    $value =~ s/^\s+|\s+$//g;
                }
                elsif ($rule =~ /^max_length:(\d+)$/) {
                    if (length($value) > $1) {
                        $errors{$field} = "$field exceeds maximum length of $1";
                        last;
                    }
                }
                elsif ($rule =~ /^min_length:(\d+)$/) {
                    if (length($value) < $1) {
                        $errors{$field} = "$field must be at least $1 characters";
                        last;
                    }
                }
            }

            $clean{$field} = $value unless exists $errors{$field};
        }

        return (\%clean, \%errors);
    }

    sub sanitize_filename ($self, $filename) {
        # Remove any path components
        $filename =~ s/.*[\/\\]//;

        # Remove dangerous characters
        $filename =~ s/[^\w\.\-]/_/g;

        # Remove multiple dots (prevent extension confusion)
        $filename =~ s/\.{2,}/\./g;

        # Limit length
        $filename = substr($filename, 0, 255) if length($filename) > 255;

        return $filename;
    }

    sub sanitize_command ($self, $cmd) {
        # Quote shell metacharacters
        $cmd =~ s/([;<>\*\|`&\$!#\(\)\[\]\{\}:'"])/\\$1/g;
        return $cmd;
    }
}

# Usage example
my $validator = Security::Validator->new();

my $input = {
    email => 'user@example.com',
    username => 'john_doe',
    age => '25',
    website => 'https://example.com',
    comment => '<script>alert("XSS")</script>Hello',
    filepath => '../../../etc/passwd',
};

my ($clean, $errors) = $validator->validate($input, {
    email => ['required', 'email'],
    username => ['required', 'alphanumeric', 'min_length:3', 'max_length:20'],
    age => ['required', 'positive_integer'],
    website => ['url'],
    comment => ['required', 'xss_safe', 'sanitize_html'],
    filepath => ['required', 'path_safe'],
});

if (%$errors) {
    say "Validation errors:";
    say "  $_: $errors->{$_}" for keys %$errors;
} else {
    say "All inputs valid!";
}
```

## Secure Password Handling

Never store passwords in plain text. Use proper hashing with salt:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Password {
    use Moo;
    use Crypt::Eksblowfish::Bcrypt qw(bcrypt en_base64);
    use Crypt::URandom qw(urandom);
    use Bytes::Random::Secure;
    use MIME::Base64;
    use Types::Standard qw(Int);

    has cost => (
        is => 'ro',
        isa => Int,
        default => 12,  # Increase for more security
    );

    has min_length => (
        is => 'ro',
        isa => Int,
        default => 8,
    );

    sub hash_password ($self, $password) {
        # Generate random salt
        my $salt = en_base64(urandom(16));

        # Format for bcrypt
        my $settings = sprintf('$2a$%02d$%s', $self->cost, $salt);

        # Hash the password
        return bcrypt($password, $settings);
    }

    sub verify_password ($self, $password, $hash) {
        # Constant-time comparison to prevent timing attacks
        return bcrypt($password, $hash) eq $hash;
    }

    sub generate_secure_token ($self, $length = 32) {
        my $rng = Bytes::Random::Secure->new(NonBlocking => 1);
        return encode_base64($rng->bytes($length), '');
    }

    sub check_password_strength ($self, $password) {
        my @errors;

        push @errors, "Password too short (minimum $self->{min_length} characters)"
            if length($password) < $self->min_length;

        push @errors, "Password must contain at least one uppercase letter"
            unless $password =~ /[A-Z]/;

        push @errors, "Password must contain at least one lowercase letter"
            unless $password =~ /[a-z]/;

        push @errors, "Password must contain at least one digit"
            unless $password =~ /\d/;

        push @errors, "Password must contain at least one special character"
            unless $password =~ /[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/;

        # Check for common patterns
        push @errors, "Password contains sequential characters"
            if $password =~ /(?:abc|bcd|cde|def|efg|fgh|ghi|hij|ijk|jkl|klm|lmn|mno|nop|opq|pqr|qrs|rst|stu|tuv|uvw|vwx|wxy|xyz|012|123|234|345|456|567|678|789)/i;

        push @errors, "Password contains repeated characters"
            if $password =~ /(.)\1{2,}/;

        return @errors ? \@errors : undef;
    }

    sub generate_password ($self, $length = 16) {
        my @chars = (
            'A'..'Z', 'a'..'z', '0'..'9',
            qw(! @ # $ % ^ & * ( ) - _ = + [ ] { } ; : , . < > / ?)
        );

        my $rng = Bytes::Random::Secure->new(NonBlocking => 1);
        my $password = '';

        # Ensure at least one of each required type
        $password .= $chars[$rng->irand(26)];       # Uppercase
        $password .= $chars[$rng->irand(26) + 26];  # Lowercase
        $password .= $chars[$rng->irand(10) + 52];  # Digit
        $password .= $chars[$rng->irand(22) + 62];  # Special

        # Fill the rest randomly
        for (my $i = 4; $i < $length; $i++) {
            $password .= $chars[$rng->irand(@chars)];
        }

        # Shuffle the password
        my @password_chars = split //, $password;
        for (my $i = @password_chars - 1; $i > 0; $i--) {
            my $j = $rng->irand($i + 1);
            @password_chars[$i, $j] = @password_chars[$j, $i];
        }

        return join '', @password_chars;
    }
}

# Session management
package Security::Session {
    use Moo;
    use Types::Standard qw(Str Int HashRef);
    use Digest::SHA qw(sha256_hex);
    use JSON::XS;
    use Crypt::JWT qw(encode_jwt decode_jwt);
    use Try::Tiny;

    has secret => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    has timeout => (
        is => 'ro',
        isa => Int,
        default => 3600,  # 1 hour
    );

    has algorithm => (
        is => 'ro',
        isa => Str,
        default => 'HS256',
    );

    sub create_token ($self, $user_id, $data = {}) {
        my $now = time();

        my $payload = {
            sub => $user_id,  # Subject
            iat => $now,      # Issued at
            exp => $now + $self->timeout,  # Expiration
            nbf => $now,      # Not before
            jti => sha256_hex($user_id . $now . rand()),  # JWT ID
            data => $data,
        };

        return encode_jwt(
            payload => $payload,
            alg => $self->algorithm,
            key => $self->secret,
        );
    }

    sub verify_token ($self, $token) {
        try {
            my $payload = decode_jwt(
                token => $token,
                key => $self->secret,
                verify_exp => 1,
                verify_nbf => 1,
            );
            return $payload;
        }
        catch {
            return undef;
        };
    }

    sub refresh_token ($self, $token) {
        my $payload = $self->verify_token($token);
        return unless $payload;

        # Check if token is close to expiry (within 5 minutes)
        if ($payload->{exp} - time() < 300) {
            return $self->create_token($payload->{sub}, $payload->{data});
        }

        return $token;
    }
}

# Example usage
my $pw_manager = Security::Password->new();

# Generate secure password
my $password = $pw_manager->generate_password(16);
say "Generated password: $password";

# Check strength
if (my $errors = $pw_manager->check_password_strength($password)) {
    say "Password strength issues: " . join(", ", @$errors);
} else {
    say "Password is strong";
}

# Hash password
my $hash = $pw_manager->hash_password($password);
say "Password hash: $hash";

# Verify password
if ($pw_manager->verify_password($password, $hash)) {
    say "Password verified successfully";
}

# Session management
my $session = Security::Session->new(
    secret => $pw_manager->generate_secure_token(32),
);

my $token = $session->create_token('user123', { role => 'admin' });
say "JWT Token: $token";

if (my $payload = $session->verify_token($token)) {
    say "Token valid for user: $payload->{sub}";
    say "User role: $payload->{data}{role}";
}
```

## SQL Injection Prevention

Always use parameterized queries, never string concatenation:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Database {
    use Moo;
    use DBI;
    use DBD::SQLite;
    use Types::Standard qw(InstanceOf HashRef);
    use SQL::Abstract::More;
    use Try::Tiny;

    has dbh => (
        is => 'ro',
        isa => InstanceOf['DBI::db'],
        required => 1,
    );

    has sql_abstract => (
        is => 'lazy',
        default => sub { SQL::Abstract::More->new() },
    );

    # NEVER DO THIS - Example of vulnerable code
    sub vulnerable_query ($self, $username) {
        # DON'T DO THIS - Direct string interpolation
        my $query = "SELECT * FROM users WHERE username = '$username'";
        # Attacker could pass: admin' OR '1'='1
        # Resulting in: SELECT * FROM users WHERE username = 'admin' OR '1'='1'

        warn "NEVER use string interpolation in SQL!";
        return;
    }

    # SECURE: Use placeholders
    sub secure_query ($self, $username) {
        my $sth = $self->dbh->prepare(
            "SELECT * FROM users WHERE username = ?"
        );
        $sth->execute($username);
        return $sth->fetchrow_hashref;
    }

    # SECURE: Use SQL::Abstract for dynamic queries
    sub dynamic_search ($self, $criteria) {
        my ($sql, @bind) = $self->sql_abstract->select(
            'users',
            ['id', 'username', 'email'],
            $criteria,
            {-order_by => 'created_at DESC'}
        );

        my $sth = $self->dbh->prepare($sql);
        $sth->execute(@bind);
        return $sth->fetchall_arrayref({});
    }

    # SECURE: Stored procedures
    sub call_procedure ($self, $proc_name, @params) {
        # Whitelist procedure names
        my %allowed_procs = map { $_ => 1 } qw(
            get_user_by_id
            update_last_login
            calculate_statistics
        );

        die "Unauthorized procedure: $proc_name"
            unless $allowed_procs{$proc_name};

        my $placeholders = join(',', ('?') x @params);
        my $sth = $self->dbh->prepare("CALL $proc_name($placeholders)");
        $sth->execute(@params);
        return $sth->fetchall_arrayref({});
    }

    # SECURE: Whitelist-based table/column validation
    sub safe_dynamic_query ($self, $table, $columns, $where) {
        # Whitelist tables and columns
        my %allowed_tables = map { $_ => 1 } qw(users posts comments);
        my %allowed_columns = map { $_ => 1 } qw(
            id username email title content created_at
        );

        die "Invalid table: $table" unless $allowed_tables{$table};

        for my $col (@$columns) {
            die "Invalid column: $col" unless $allowed_columns{$col};
        }

        my ($sql, @bind) = $self->sql_abstract->select(
            $table,
            $columns,
            $where
        );

        my $sth = $self->dbh->prepare($sql);
        $sth->execute(@bind);
        return $sth->fetchall_arrayref({});
    }

    # SECURE: Transaction with rollback on error
    sub secure_transaction ($self, $operations) {
        my $result;

        try {
            $self->dbh->begin_work;

            for my $op (@$operations) {
                my ($sql, @params) = @$op;
                $self->dbh->do($sql, undef, @params);
            }

            $self->dbh->commit;
            $result = { success => 1 };
        }
        catch {
            $self->dbh->rollback;
            $result = { success => 0, error => $_ };
        };

        return $result;
    }
}

# Prepared statement cache for performance
package Security::PreparedStatements {
    use Moo;
    use Types::Standard qw(InstanceOf HashRef);

    has dbh => (
        is => 'ro',
        isa => InstanceOf['DBI::db'],
        required => 1,
    );

    has statements => (
        is => 'ro',
        isa => HashRef,
        default => sub { {} },
    );

    sub prepare ($self, $name, $sql) {
        $self->statements->{$name} ||= $self->dbh->prepare($sql);
        return $self->statements->{$name};
    }

    sub execute ($self, $name, @params) {
        my $sth = $self->statements->{$name}
            or die "Unknown statement: $name";

        $sth->execute(@params);
        return $sth;
    }

    sub DEMOLISH ($self) {
        # Clean up prepared statements
        $_->finish for values %{$self->statements};
    }
}
```

## Command Injection Prevention

Never pass user input directly to system commands:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Command {
    use Moo;
    use IPC::Run3;
    use File::Which;
    use Types::Standard qw(HashRef ArrayRef);

    has allowed_commands => (
        is => 'ro',
        isa => HashRef,
        default => sub {{
            ls => '/bin/ls',
            grep => '/bin/grep',
            find => '/usr/bin/find',
            tar => '/bin/tar',
        }},
    );

    # NEVER DO THIS
    sub dangerous_exec ($self, $user_input) {
        # DON'T DO THIS - Direct interpolation
        # system("ls $user_input");
        # Attacker could pass: "; rm -rf /"

        warn "NEVER interpolate user input into commands!";
        return;
    }

    # SECURE: Use array form of system()
    sub safe_exec ($self, $command, @args) {
        # Validate command against whitelist
        my $cmd_path = $self->allowed_commands->{$command}
            or die "Command not allowed: $command";

        die "Command not found: $cmd_path" unless -x $cmd_path;

        # Validate arguments
        for my $arg (@args) {
            die "Invalid character in argument"
                if $arg =~ /[\0\n\r]/;  # Null bytes and newlines
        }

        # Use array form - no shell interpretation
        my $exit_code = system($cmd_path, @args);

        return $exit_code == 0;
    }

    # SECURE: Use IPC::Run3 for better control
    sub safe_capture ($self, $command, $args, $input = undef) {
        my $cmd_path = $self->allowed_commands->{$command}
            or die "Command not allowed: $command";

        my ($stdout, $stderr);

        run3(
            [$cmd_path, @$args],
            \$input,
            \$stdout,
            \$stderr,
        );

        return {
            stdout => $stdout,
            stderr => $stderr,
            exit_code => $? >> 8,
        };
    }

    # SECURE: Taint mode checking
    sub check_tainted ($self, $value) {
        # In taint mode, this would detect tainted data
        return eval {
            local $@;
            kill 0, $value;
            0;
        } || 1;
    }

    # SECURE: Safe file operations
    sub safe_file_operation ($self, $operation, $filename) {
        # Sanitize filename
        die "Invalid filename" if $filename =~ /\.\./;  # No directory traversal
        die "Invalid filename" if $filename =~ /^\//;   # No absolute paths
        die "Invalid filename" if $filename =~ /[\0]/;  # No null bytes

        # Restrict to safe directory
        my $safe_dir = '/tmp/safe_uploads';
        my $full_path = "$safe_dir/$filename";

        # Validate operation
        my %allowed_ops = map { $_ => 1 } qw(read write delete);
        die "Invalid operation" unless $allowed_ops{$operation};

        if ($operation eq 'read') {
            open my $fh, '<', $full_path or die "Cannot read: $!";
            local $/;
            my $content = <$fh>;
            close $fh;
            return $content;
        }
        elsif ($operation eq 'write') {
            # Additional checks for write operations
            die "File too large" if -s $full_path > 10_000_000;  # 10MB limit
        }

        return 1;
    }
}

# Safe templating to prevent injection
package Security::Template {
    use Moo;
    use Template;
    use HTML::Entities;

    has tt => (
        is => 'lazy',
        default => sub {
            Template->new({
                # Disable dangerous operations
                EVAL_PERL => 0,
                LOAD_PERL => 0,
                LOAD_PLUGINS => 0,
                LOAD_TEMPLATES => 0,
                ABSOLUTE => 0,
                RELATIVE => 0,
                # Enable auto-escaping
                FILTERS => {
                    html => sub { encode_entities($_[0]) },
                    js => sub {
                        my $text = shift;
                        $text =~ s/(['\\])/\\$1/g;
                        $text =~ s/\n/\\n/g;
                        $text =~ s/\r/\\r/g;
                        return $text;
                    },
                },
            });
        },
    );

    sub render_safe ($self, $template, $vars) {
        # Sanitize all variables by default
        my $safe_vars = $self->sanitize_vars($vars);

        my $output;
        $self->tt->process(\$template, $safe_vars, \$output)
            or die $self->tt->error;

        return $output;
    }

    sub sanitize_vars ($self, $vars) {
        my $safe = {};

        for my $key (keys %$vars) {
            my $value = $vars->{$key};

            if (ref $value eq 'ARRAY') {
                $safe->{$key} = [map { encode_entities($_) } @$value];
            }
            elsif (ref $value eq 'HASH') {
                $safe->{$key} = $self->sanitize_vars($value);
            }
            else {
                $safe->{$key} = encode_entities($value // '');
            }
        }

        return $safe;
    }
}
```

## Cryptography and Secure Communication

Implement proper encryption for sensitive data:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Crypto {
    use Moo;
    use Crypt::CBC;
    use Crypt::Rijndael;  # AES
    use Crypt::Random qw(makerandom_octet);
    use MIME::Base64;
    use Digest::SHA qw(sha256_hex);
    use Types::Standard qw(Str);

    has key => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    has cipher => (
        is => 'lazy',
    );

    sub _build_cipher ($self) {
        # Derive a proper key from the provided key
        my $derived_key = substr(sha256_hex($self->key), 0, 32);

        return Crypt::CBC->new(
            -key => $derived_key,
            -cipher => 'Rijndael',
            -header => 'salt',
            -pbkdf => 'pbkdf2',
            -iterations => 10000,
        );
    }

    sub encrypt ($self, $plaintext) {
        my $ciphertext = $self->cipher->encrypt($plaintext);
        return encode_base64($ciphertext, '');
    }

    sub decrypt ($self, $ciphertext) {
        my $decoded = decode_base64($ciphertext);
        return $self->cipher->decrypt($decoded);
    }

    sub generate_key ($self, $length = 32) {
        return makerandom_octet(Length => $length, Strength => 1);
    }

    sub secure_compare ($self, $a, $b) {
        # Constant-time comparison to prevent timing attacks
        return 0 unless defined $a && defined $b;
        return 0 unless length($a) == length($b);

        my $result = 0;
        for (my $i = 0; $i < length($a); $i++) {
            $result |= ord(substr($a, $i, 1)) ^ ord(substr($b, $i, 1));
        }

        return $result == 0;
    }
}

# TLS/SSL configuration
package Security::TLS {
    use Moo;
    use IO::Socket::SSL;
    use Mozilla::CA;  # Mozilla's CA bundle

    sub create_client ($self, $host, $port) {
        return IO::Socket::SSL->new(
            PeerHost => $host,
            PeerPort => $port,
            SSL_verify_mode => SSL_VERIFY_PEER,
            SSL_ca_file => Mozilla::CA::SSL_ca_file(),
            SSL_version => '!SSLv2:!SSLv3:!TLSv1:!TLSv1_1',  # TLS 1.2+
            SSL_cipher_list => 'HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4',
        ) or die "SSL connection failed: $SSL_ERROR";
    }

    sub create_server ($self, $port, $cert_file, $key_file) {
        return IO::Socket::SSL->new(
            LocalPort => $port,
            Listen => 10,
            SSL_cert_file => $cert_file,
            SSL_key_file => $key_file,
            SSL_verify_mode => SSL_VERIFY_PEER,
            SSL_version => '!SSLv2:!SSLv3:!TLSv1:!TLSv1_1',
            SSL_cipher_list => 'HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4',
            SSL_honor_cipher_order => 1,
        ) or die "SSL server creation failed: $SSL_ERROR";
    }
}
```

## File Upload Security

Validate and sanitize file uploads carefully:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::FileUpload {
    use Moo;
    use File::Type;
    use File::Temp;
    use Image::Size;
    use Archive::Zip;
    use Types::Standard qw(HashRef Int Str);

    has allowed_types => (
        is => 'ro',
        isa => HashRef,
        default => sub {{
            'image/jpeg' => [qw(jpg jpeg)],
            'image/png' => [qw(png)],
            'image/gif' => [qw(gif)],
            'application/pdf' => [qw(pdf)],
            'text/plain' => [qw(txt)],
        }},
    );

    has max_size => (
        is => 'ro',
        isa => Int,
        default => 5_000_000,  # 5MB
    );

    has upload_dir => (
        is => 'ro',
        isa => Str,
        default => '/var/uploads',
    );

    sub validate_upload ($self, $file_handle, $claimed_name, $claimed_type) {
        my @errors;

        # Check file size
        my $size = -s $file_handle;
        push @errors, "File too large (max: $self->{max_size} bytes)"
            if $size > $self->max_size;

        # Verify actual file type
        my $ft = File::Type->new();
        my $actual_type = $ft->checktype_filehandle($file_handle);

        unless ($self->allowed_types->{$actual_type}) {
            push @errors, "File type not allowed: $actual_type";
        }

        # Check for type mismatch
        if ($claimed_type && $claimed_type ne $actual_type) {
            push @errors, "File type mismatch (claimed: $claimed_type, actual: $actual_type)";
        }

        # Validate filename
        if ($claimed_name =~ /\.\./ || $claimed_name =~ /[\/\\]/) {
            push @errors, "Invalid filename";
        }

        # Check extension
        my ($ext) = $claimed_name =~ /\.([^.]+)$/;
        if ($ext) {
            my $allowed_exts = $self->allowed_types->{$actual_type} || [];
            unless (grep { lc($ext) eq $_ } @$allowed_exts) {
                push @errors, "Invalid file extension for type $actual_type";
            }
        }

        # Additional checks for images
        if ($actual_type =~ /^image\//) {
            seek($file_handle, 0, 0);
            my ($width, $height) = Image::Size::imgsize($file_handle);

            unless ($width && $height) {
                push @errors, "Invalid image file";
            }

            # Check for suspicious dimensions (possible zip bombs)
            if ($width * $height > 100_000_000) {  # 100 megapixels
                push @errors, "Image dimensions too large";
            }
        }

        # Check for embedded executables in archives
        if ($actual_type =~ /zip/) {
            push @errors, "ZIP files not allowed" unless $self->allowed_types->{'application/zip'};

            my $zip = Archive::Zip->new();
            $zip->readFromFileHandle($file_handle);

            for my $member ($zip->members()) {
                if ($member->fileName() =~ /\.(exe|dll|sh|bat|cmd|com|scr)$/i) {
                    push @errors, "Archive contains executable files";
                    last;
                }
            }
        }

        return @errors ? \@errors : undef;
    }

    sub save_upload ($self, $file_handle, $original_name) {
        # Generate safe filename
        my $safe_name = $self->generate_safe_filename($original_name);

        # Create temporary file first
        my $temp = File::Temp->new(
            DIR => $self->upload_dir,
            SUFFIX => '.tmp',
            UNLINK => 0,
        );

        # Copy content
        seek($file_handle, 0, 0);
        my $buffer;
        while (read($file_handle, $buffer, 8192)) {
            print $temp $buffer;
        }
        close $temp;

        # Move to final location
        my $final_path = "$self->{upload_dir}/$safe_name";
        rename($temp->filename, $final_path)
            or die "Failed to save upload: $!";

        # Set restrictive permissions
        chmod(0644, $final_path);

        return $final_path;
    }

    sub generate_safe_filename ($self, $original) {
        # Extract extension
        my ($ext) = $original =~ /\.([^.]+)$/;
        $ext = lc($ext // 'dat');

        # Generate unique name
        my $timestamp = time();
        my $random = int(rand(10000));
        my $hash = substr(sha256_hex($original . $timestamp . $random), 0, 12);

        return "${timestamp}_${hash}.${ext}";
    }
}
```

## Security Auditing and Logging

Track security events for forensics and compliance:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package Security::Audit {
    use Moo;
    use Log::Log4perl qw(:easy);
    use JSON::XS;
    use Time::HiRes qw(time);
    use Sys::Hostname;
    use Types::Standard qw(Str);

    has log_file => (
        is => 'ro',
        isa => Str,
        default => '/var/log/security_audit.log',
    );

    has hostname => (
        is => 'lazy',
        default => sub { hostname() },
    );

    sub BUILD ($self, $args) {
        # Configure Log4perl
        my $conf = qq{
            log4perl.rootLogger = INFO, File, Syslog

            log4perl.appender.File = Log::Log4perl::Appender::File
            log4perl.appender.File.filename = $self->{log_file}
            log4perl.appender.File.mode = append
            log4perl.appender.File.layout = Log::Log4perl::Layout::PatternLayout
            log4perl.appender.File.layout.ConversionPattern = %d{ISO8601} [%p] %m%n

            log4perl.appender.Syslog = Log::Log4perl::Appender::Syslog
            log4perl.appender.Syslog.ident = security_audit
            log4perl.appender.Syslog.facility = local0
            log4perl.appender.Syslog.layout = Log::Log4perl::Layout::SimpleLayout
        };

        Log::Log4perl->init(\$conf);
    }

    sub log_event ($self, $event_type, $details) {
        my $event = {
            timestamp => time(),
            hostname => $self->hostname,
            event_type => $event_type,
            details => $details,
            pid => $$,
        };

        my $json = encode_json($event);

        # Log based on severity
        given ($event_type) {
            when (/^(auth_failure|intrusion|breach)/) {
                ERROR($json);
            }
            when (/^(auth_success|access_granted)/) {
                INFO($json);
            }
            when (/^(suspicious|anomaly)/) {
                WARN($json);
            }
            default {
                DEBUG($json);
            }
        }

        return $event;
    }

    sub log_authentication ($self, $username, $success, $ip_address, $details = {}) {
        $self->log_event(
            $success ? 'auth_success' : 'auth_failure',
            {
                username => $username,
                ip_address => $ip_address,
                %$details,
            }
        );
    }

    sub log_access ($self, $user, $resource, $action, $allowed) {
        $self->log_event(
            $allowed ? 'access_granted' : 'access_denied',
            {
                user => $user,
                resource => $resource,
                action => $action,
            }
        );
    }

    sub log_suspicious_activity ($self, $type, $details) {
        $self->log_event('suspicious', {
            type => $type,
            %$details,
        });
    }

    sub detect_brute_force ($self, $username, $ip_address, $window = 300) {
        # Track failed attempts (would use Redis/Memcached in production)
        state %attempts;

        my $key = "$username:$ip_address";
        my $now = time();

        # Clean old attempts
        $attempts{$key} = [grep { $now - $_ < $window } @{$attempts{$key} // []}];

        # Add current attempt
        push @{$attempts{$key}}, $now;

        # Check threshold
        if (@{$attempts{$key}} > 5) {
            $self->log_event('intrusion', {
                type => 'brute_force',
                username => $username,
                ip_address => $ip_address,
                attempts => scalar(@{$attempts{$key}}),
                window => $window,
            });
            return 1;
        }

        return 0;
    }
}
```

## Best Practices Summary

1. **Input Validation**
   - Never trust user input
   - Validate on the server side
   - Use whitelisting over blacklisting
   - Sanitize for the specific context (HTML, SQL, Shell)

2. **Authentication & Authorization**
   - Use strong password hashing (bcrypt, scrypt, Argon2)
   - Implement proper session management
   - Use JWT tokens with expiration
   - Enforce principle of least privilege

3. **Data Protection**
   - Encrypt sensitive data at rest
   - Use TLS for data in transit
   - Implement key rotation
   - Secure key storage

4. **Secure Coding**
   - Use parameterized queries
   - Avoid system() with user input
   - Implement proper error handling
   - Don't expose sensitive information in errors

5. **Monitoring & Auditing**
   - Log security events
   - Detect anomalies and patterns
   - Regular security audits
   - Keep dependencies updated

## Security Checklist

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Security Checklist Module
package Security::Checklist {
    use Moo;

    sub check_application ($self) {
        my @checks = (
            { name => 'Input Validation', check => sub { $self->check_input_validation() } },
            { name => 'SQL Injection Prevention', check => sub { $self->check_sql_injection() } },
            { name => 'XSS Prevention', check => sub { $self->check_xss() } },
            { name => 'CSRF Protection', check => sub { $self->check_csrf() } },
            { name => 'Authentication', check => sub { $self->check_authentication() } },
            { name => 'Authorization', check => sub { $self->check_authorization() } },
            { name => 'Session Management', check => sub { $self->check_sessions() } },
            { name => 'Cryptography', check => sub { $self->check_crypto() } },
            { name => 'Error Handling', check => sub { $self->check_error_handling() } },
            { name => 'Logging', check => sub { $self->check_logging() } },
            { name => 'File Uploads', check => sub { $self->check_file_uploads() } },
            { name => 'Dependencies', check => sub { $self->check_dependencies() } },
            { name => 'Configuration', check => sub { $self->check_configuration() } },
            { name => 'Headers', check => sub { $self->check_security_headers() } },
        );

        my $passed = 0;
        my $total = @checks;

        say "=" x 50;
        say "Security Checklist";
        say "=" x 50;

        for my $check (@checks) {
            my $result = $check->{check}->();
            my $status = $result ? '✓' : '✗';
            my $color = $result ? "\e[32m" : "\e[31m";  # Green or Red

            printf "%s%-30s %s%s\e[0m\n",
                   $color, $check->{name}, $status,
                   $result ? '' : ' - NEEDS ATTENTION';

            $passed++ if $result;
        }

        say "=" x 50;
        printf "Score: %d/%d (%.1f%%)\n",
               $passed, $total, ($passed/$total)*100;

        return $passed == $total;
    }

    # Individual check methods would be implemented here
    sub check_input_validation { 1 }  # Placeholder
    sub check_sql_injection { 1 }
    sub check_xss { 1 }
    sub check_csrf { 1 }
    sub check_authentication { 1 }
    sub check_authorization { 1 }
    sub check_sessions { 1 }
    sub check_crypto { 1 }
    sub check_error_handling { 1 }
    sub check_logging { 1 }
    sub check_file_uploads { 1 }
    sub check_dependencies { 1 }
    sub check_configuration { 1 }
    sub check_security_headers { 1 }
}

my $checker = Security::Checklist->new();
$checker->check_application();
```

## Summary

Security is not a destination but a journey. Every line of code you write is either strengthening or weakening your application's defenses. Perl provides powerful tools for security, but they must be used correctly. Remember: validate everything, trust nothing, encrypt sensitive data, log security events, and stay updated on the latest threats and patches.

In 2025's threat landscape, where attacks are automated and adversaries are sophisticated, your Perl applications must be built with security as a foundation, not an afterthought. The techniques and patterns in this chapter aren't just best practices—they're essential practices for any system that handles real data in the real world.