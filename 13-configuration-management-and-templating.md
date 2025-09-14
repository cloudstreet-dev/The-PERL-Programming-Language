# Chapter 13: Configuration Management and Templating

> "Hard-coding values is like welding your car's steering wheel - it works great until you need to turn." - DevOps Wisdom

Configuration and templates are the bridge between your code and the real world. They're what make your applications flexible, maintainable, and deployable across different environments. This chapter covers everything from simple config files to sophisticated templating systems that can generate anything from HTML to system configurations.

## Configuration File Formats

### INI Files with Config::Tiny

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Config::Tiny;

# Read INI file
my $config = Config::Tiny->read('app.ini');

# Access values
my $db_host = $config->{database}{host};
my $db_port = $config->{database}{port};
my $log_level = $config->{logging}{level};

# Modify and write
$config->{database}{port} = 3307;
$config->{new_section}{key} = 'value';
$config->write('app.ini');

# Create from scratch
my $new_config = Config::Tiny->new;
$new_config->{server} = {
    host => 'localhost',
    port => 8080,
    workers => 4,
};
$new_config->{features} = {
    cache => 'enabled',
    debug => 'disabled',
};
$new_config->write('new_config.ini');

# Example INI file:
# [database]
# host = localhost
# port = 3306
# name = myapp
#
# [logging]
# level = info
# file = /var/log/myapp.log
```

### YAML Configuration

```perl
use YAML::XS qw(LoadFile DumpFile);

# Load YAML config
my $config = LoadFile('config.yaml');

# Access nested structures
my $redis_host = $config->{cache}{redis}{host};
my $features = $config->{features};

# Save config
DumpFile('config_backup.yaml', $config);

# Multi-document YAML
use YAML::XS qw(Load);

my $yaml = <<'END_YAML';
---
environment: development
debug: true
---
environment: production
debug: false
END_YAML

my @configs = Load($yaml);

# Example YAML structure:
# database:
#   primary:
#     host: db1.example.com
#     port: 5432
#     name: app_prod
#   replica:
#     - host: db2.example.com
#       port: 5432
#     - host: db3.example.com
#       port: 5432
#
# features:
#   - authentication
#   - caching
#   - monitoring
#
# settings:
#   timeout: 30
#   retries: 3
```

### JSON Configuration

```perl
use JSON::XS;
use Path::Tiny;

# Read JSON config
sub load_json_config {
    my ($file) = @_;

    my $content = path($file)->slurp_utf8;
    my $config = decode_json($content);

    # Support comments (non-standard)
    $content =~ s/^\s*\/\/.*$//mg;  # Remove // comments
    $content =~ s/\/\*.*?\*\///sg;  # Remove /* */ comments

    return decode_json($content);
}

# Save with pretty formatting
sub save_json_config {
    my ($file, $config) = @_;

    my $json = JSON::XS->new->utf8->pretty->canonical;
    path($file)->spew_utf8($json->encode($config));
}

# Merge configs
sub merge_configs {
    my ($base, $override) = @_;

    my %merged = %$base;
    for my $key (keys %$override) {
        if (ref $override->{$key} eq 'HASH' && ref $merged{$key} eq 'HASH') {
            $merged{$key} = merge_configs($merged{$key}, $override->{$key});
        } else {
            $merged{$key} = $override->{$key};
        }
    }

    return \%merged;
}
```

## Advanced Configuration Management

### Environment-Based Configuration

```perl
package Config::Environment;
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

sub new($class, $base_dir = './config') {
    return bless {
        base_dir => $base_dir,
        env => $ENV{APP_ENV} // 'development',
        cache => {},
    }, $class;
}

sub load($self) {
    my $env = $self->{env};

    # Load base config
    my $base = $self->load_file('default');

    # Load environment-specific config
    my $env_config = $self->load_file($env);

    # Merge configs
    my $config = $self->merge($base, $env_config);

    # Override with environment variables
    $self->apply_env_overrides($config);

    return $config;
}

sub load_file($self, $name) {
    my $file = "$self->{base_dir}/$name.yaml";
    return {} unless -f $file;

    return LoadFile($file);
}

sub merge($self, $base, $override) {
    # Deep merge implementation
    return merge_configs($base, $override);
}

sub apply_env_overrides($self, $config) {
    # Override config with environment variables
    # APP_DATABASE_HOST overrides $config->{database}{host}

    for my $env_key (keys %ENV) {
        next unless $env_key =~ /^APP_/;

        my @path = map { lc } split /_/, $env_key;
        shift @path;  # Remove 'APP'

        my $ref = $config;
        for my $i (0..$#path-1) {
            $ref->{$path[$i]} //= {};
            $ref = $ref->{$path[$i]};
        }
        $ref->{$path[-1]} = $ENV{$env_key};
    }
}

sub get($self, $path, $default = undef) {
    my @parts = split /\./, $path;
    my $value = $self->{cache};

    for my $part (@parts) {
        return $default unless ref $value eq 'HASH';
        $value = $value->{$part};
        return $default unless defined $value;
    }

    return $value;
}

# Usage
my $config = Config::Environment->new('./config');
my $settings = $config->load();

my $db_host = $config->get('database.host', 'localhost');
my $timeout = $config->get('app.timeout', 30);
```

### Configuration Validation

```perl
package Config::Validator;
use Modern::Perl '2023';

sub new {
    my ($class) = @_;
    return bless {
        rules => {},
        errors => [],
    }, $class;
}

sub add_rule {
    my ($self, $path, $rule) = @_;
    $self->{rules}{$path} = $rule;
    return $self;
}

sub validate {
    my ($self, $config) = @_;
    $self->{errors} = [];

    for my $path (keys %{$self->{rules}}) {
        my $value = $self->get_value($config, $path);
        my $rule = $self->{rules}{$path};

        $self->check_rule($path, $value, $rule);
    }

    return @{$self->{errors}} == 0;
}

sub get_value {
    my ($self, $config, $path) = @_;

    my @parts = split /\./, $path;
    my $value = $config;

    for my $part (@parts) {
        return undef unless ref $value eq 'HASH';
        $value = $value->{$part};
    }

    return $value;
}

sub check_rule {
    my ($self, $path, $value, $rule) = @_;

    # Required check
    if ($rule->{required} && !defined $value) {
        push @{$self->{errors}}, "$path is required";
        return;
    }

    return unless defined $value;

    # Type check
    if ($rule->{type}) {
        my $type = ref $value || 'scalar';
        if ($rule->{type} eq 'int' && $value !~ /^\d+$/) {
            push @{$self->{errors}}, "$path must be an integer";
        } elsif ($rule->{type} eq 'array' && $type ne 'ARRAY') {
            push @{$self->{errors}}, "$path must be an array";
        } elsif ($rule->{type} eq 'hash' && $type ne 'HASH') {
            push @{$self->{errors}}, "$path must be a hash";
        }
    }

    # Range check
    if (defined $rule->{min} && $value < $rule->{min}) {
        push @{$self->{errors}}, "$path must be at least $rule->{min}";
    }
    if (defined $rule->{max} && $value > $rule->{max}) {
        push @{$self->{errors}}, "$path must be at most $rule->{max}";
    }

    # Pattern check
    if ($rule->{pattern} && $value !~ /$rule->{pattern}/) {
        push @{$self->{errors}}, "$path doesn't match required pattern";
    }

    # Custom validator
    if ($rule->{validator}) {
        my $error = $rule->{validator}->($value);
        push @{$self->{errors}}, "$path: $error" if $error;
    }
}

sub errors {
    my ($self) = @_;
    return @{$self->{errors}};
}

# Usage
my $validator = Config::Validator->new();

$validator->add_rule('database.host', {
    required => 1,
    pattern => qr/^[\w\-\.]+$/,
});

$validator->add_rule('database.port', {
    required => 1,
    type => 'int',
    min => 1,
    max => 65535,
});

$validator->add_rule('app.workers', {
    type => 'int',
    min => 1,
    max => 100,
    validator => sub {
        my $val = shift;
        return "must be even" if $val % 2 != 0;
        return undef;
    },
});

if (!$validator->validate($config)) {
    die "Configuration errors:\n  " . join("\n  ", $validator->errors());
}
```

## Template Toolkit

### Basic Template Usage

```perl
use Template;

# Create template processor
my $tt = Template->new({
    INCLUDE_PATH => './templates',
    INTERPOLATE  => 1,
    POST_CHOMP   => 1,
    PRE_PROCESS  => 'header.tt',
    POST_PROCESS => 'footer.tt',
    ENCODING     => 'utf8',
});

# Process template
my $vars = {
    title => 'My Page',
    user => {
        name => 'Alice',
        email => 'alice@example.com',
    },
    items => [
        { name => 'Item 1', price => 10.50 },
        { name => 'Item 2', price => 25.00 },
    ],
};

my $output;
$tt->process('page.tt', $vars, \$output) or die $tt->error;
print $output;

# Template file (page.tt):
# [% BLOCK page %]
# <h1>[% title %]</h1>
# <p>Welcome, [% user.name %]!</p>
#
# <ul>
# [% FOREACH item IN items %]
#   <li>[% item.name %] - $[% item.price | format('%.2f') %]</li>
# [% END %]
# </ul>
#
# [% total = 0 %]
# [% FOREACH item IN items %]
#   [% total = total + item.price %]
# [% END %]
# <p>Total: $[% total | format('%.2f') %]</p>
# [% END %]
```

### Advanced Template Features

```perl
# Custom filters
my $tt = Template->new({
    FILTERS => {
        markdown => sub {
            my $text = shift;
            # Convert markdown to HTML
            return markdown_to_html($text);
        },
        truncate => [sub {
            my ($context, $length) = @_;
            return sub {
                my $text = shift;
                return substr($text, 0, $length) . '...';
            };
        }, 1],  # 1 = takes arguments
    },
});

# Virtual methods
my $tt = Template->new({
    STASH => Template::Stash->new({
        VMETHODS => {
            list => {
                sum => sub {
                    my $list = shift;
                    my $sum = 0;
                    $sum += $_ for @$list;
                    return $sum;
                },
            },
            hash => {
                keys_sorted => sub {
                    my $hash = shift;
                    return [ sort keys %$hash ];
                },
            },
        },
    }),
});

# Template with custom filters and methods:
# [% content | markdown %]
# [% description | truncate(100) %]
# [% numbers.sum %]
# [% FOREACH key IN data.keys_sorted %]
#   [% key %]: [% data.$key %]
# [% END %]

# Macros and includes
# macros.tt:
# [% MACRO button(text, url, class='btn-default') BLOCK %]
#   <a href="[% url %]" class="btn [% class %]">[% text %]</a>
# [% END %]
#
# [% MACRO form_field(name, label, type='text', value='') BLOCK %]
#   <div class="form-group">
#     <label for="[% name %]">[% label %]</label>
#     <input type="[% type %]" name="[% name %]" value="[% value %]">
#   </div>
# [% END %]

# Using macros:
# [% INCLUDE macros.tt %]
# [% button('Click Me', '/action', 'btn-primary') %]
# [% form_field('username', 'Username') %]
# [% form_field('password', 'Password', 'password') %]
```

## Text::Template for Simple Templating

```perl
use Text::Template;

# Simple variable substitution
my $template = Text::Template->new(
    TYPE => 'STRING',
    SOURCE => 'Hello {$name}, your balance is ${$balance}.'
);

my $result = $template->fill_in(HASH => {
    name => 'Bob',
    balance => 100.50,
});

# Templates with Perl code
my $template = Text::Template->new(
    TYPE => 'FILE',
    SOURCE => 'report.tmpl',
);

my $output = $template->fill_in(HASH => {
    users => \@users,
    generate_table => \&generate_table,
});

# Template file with embedded Perl:
# Report for {scalar(localtime())}
#
# Users:
# { foreach my $user (@users) {
#     $OUT .= "  - $user->{name} ($user->{email})\n";
#   }
# }
#
# Statistics:
# { $generate_table->(\@users) }
```

## Configuration Templates

### Generating System Configs

```perl
package ConfigGenerator;
use Modern::Perl '2023';
use Template;

sub new {
    my ($class, $template_dir) = @_;

    return bless {
        tt => Template->new({
            INCLUDE_PATH => $template_dir,
            ABSOLUTE => 1,
        }),
    }, $class;
}

sub generate_nginx_config {
    my ($self, $config) = @_;

    my $template = <<'END_TEMPLATE';
server {
    listen [% port %];
    server_name [% server_name %];

    root [% document_root %];
    index index.html index.php;

[% IF ssl_enabled %]
    listen [% ssl_port %] ssl;
    ssl_certificate [% ssl_cert %];
    ssl_certificate_key [% ssl_key %];
[% END %]

[% FOREACH location IN locations %]
    location [% location.path %] {
[% IF location.proxy %]
        proxy_pass [% location.proxy %];
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
[% ELSIF location.alias %]
        alias [% location.alias %];
[% END %]
    }
[% END %]

[% IF php_enabled %]
    location ~ \.php$ {
        fastcgi_pass [% php_backend %];
        fastcgi_index index.php;
        include fastcgi_params;
    }
[% END %]
}
END_TEMPLATE

    my $output;
    $self->{tt}->process(\$template, $config, \$output)
        or die $self->{tt}->error;

    return $output;
}

sub generate_systemd_service {
    my ($self, $config) = @_;

    my $template = <<'END_TEMPLATE';
[Unit]
Description=[% description %]
After=network.target[% IF after %] [% after %][% END %]

[Service]
Type=[% type || 'simple' %]
User=[% user %]
Group=[% group || user %]
WorkingDirectory=[% working_directory %]
ExecStart=[% exec_start %]
[% IF exec_stop %]ExecStop=[% exec_stop %][% END %]
Restart=[% restart || 'on-failure' %]
RestartSec=[% restart_sec || '5' %]

[% FOREACH env IN environment %]
Environment="[% env.key %]=[% env.value %]"
[% END %]

[% IF limits %]
[% FOREACH limit IN limits %]
Limit[% limit.type %]=[% limit.value %]
[% END %]
[% END %]

[Install]
WantedBy=multi-user.target
END_TEMPLATE

    my $output;
    $self->{tt}->process(\$template, $config, \$output)
        or die $self->{tt}->error;

    return $output;
}

# Usage
my $gen = ConfigGenerator->new('./templates');

# Generate nginx config
my $nginx_config = $gen->generate_nginx_config({
    port => 80,
    server_name => 'example.com',
    document_root => '/var/www/html',
    ssl_enabled => 1,
    ssl_port => 443,
    ssl_cert => '/etc/ssl/certs/example.com.crt',
    ssl_key => '/etc/ssl/private/example.com.key',
    locations => [
        { path => '/api', proxy => 'http://localhost:3000' },
        { path => '/static', alias => '/var/www/static' },
    ],
    php_enabled => 1,
    php_backend => 'unix:/var/run/php/php7.4-fpm.sock',
});

# Generate systemd service
my $service_config = $gen->generate_systemd_service({
    description => 'My Application',
    user => 'app',
    working_directory => '/opt/myapp',
    exec_start => '/opt/myapp/bin/start.sh',
    environment => [
        { key => 'NODE_ENV', value => 'production' },
        { key => 'PORT', value => '3000' },
    ],
    limits => [
        { type => 'NOFILE', value => '65536' },
    ],
});
```

## Dynamic Report Generation

### HTML Report Generator

```perl
package ReportGenerator;
use Modern::Perl '2023';
use Template;
use Time::Piece;

sub new {
    my ($class) = @_;

    return bless {
        tt => Template->new({
            INCLUDE_PATH => './templates',
            WRAPPER => 'layout.tt',
        }),
    }, $class;
}

sub generate_report {
    my ($self, $data, $type) = @_;

    my $template = $self->get_template($type);
    my $vars = $self->prepare_data($data, $type);

    my $output;
    $self->{tt}->process($template, $vars, \$output)
        or die $self->{tt}->error;

    return $output;
}

sub prepare_data {
    my ($self, $data, $type) = @_;

    my $vars = {
        title => ucfirst($type) . ' Report',
        generated => localtime->strftime('%Y-%m-%d %H:%M:%S'),
        data => $data,
    };

    # Type-specific preparation
    if ($type eq 'sales') {
        $vars->{total} = $self->calculate_total($data);
        $vars->{chart_data} = $self->prepare_chart_data($data);
    } elsif ($type eq 'inventory') {
        $vars->{low_stock} = $self->find_low_stock($data);
        $vars->{categories} = $self->group_by_category($data);
    }

    return $vars;
}

sub get_template {
    my ($self, $type) = @_;
    return "reports/${type}.tt";
}

sub calculate_total {
    my ($self, $data) = @_;
    my $total = 0;
    $total += $_->{amount} for @$data;
    return $total;
}

sub prepare_chart_data {
    my ($self, $data) = @_;

    my %by_date;
    for my $item (@$data) {
        $by_date{$item->{date}} += $item->{amount};
    }

    return [
        map { { date => $_, amount => $by_date{$_} } }
        sort keys %by_date
    ];
}

# Template example (reports/sales.tt):
# [% WRAPPER layout.tt title = title %]
#
# <h2>Sales Report</h2>
# <p>Generated: [% generated %]</p>
#
# <table class="table">
#   <thead>
#     <tr>
#       <th>Date</th>
#       <th>Product</th>
#       <th>Quantity</th>
#       <th>Amount</th>
#     </tr>
#   </thead>
#   <tbody>
#   [% FOREACH item IN data %]
#     <tr>
#       <td>[% item.date %]</td>
#       <td>[% item.product %]</td>
#       <td>[% item.quantity %]</td>
#       <td>$[% item.amount | format('%.2f') %]</td>
#     </tr>
#   [% END %]
#   </tbody>
#   <tfoot>
#     <tr>
#       <th colspan="3">Total</th>
#       <th>$[% total | format('%.2f') %]</th>
#     </tr>
#   </tfoot>
# </table>
#
# <div id="chart" data-chart='[% chart_data | json %]'></div>
#
# [% END %]
```

## Real-World Example: Multi-Environment Deployment

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use YAML::XS qw(LoadFile DumpFile);
use Template;
use Path::Tiny;
use Getopt::Long;

package DeploymentManager;

sub new {
    my ($class, $config_dir) = @_;

    return bless {
        config_dir => $config_dir,
        tt => Template->new({
            INCLUDE_PATH => "$config_dir/templates",
        }),
    }, $class;
}

sub deploy {
    my ($self, $environment, $version) = @_;

    # Load environment config
    my $config = $self->load_config($environment);
    $config->{version} = $version;
    $config->{environment} = $environment;

    # Generate deployment files
    $self->generate_docker_compose($config);
    $self->generate_env_file($config);
    $self->generate_nginx_config($config);
    $self->generate_deploy_script($config);

    say "Deployment files generated for $environment (v$version)";
}

sub load_config {
    my ($self, $environment) = @_;

    # Load base config
    my $base = LoadFile("$self->{config_dir}/base.yaml");

    # Load environment-specific config
    my $env_file = "$self->{config_dir}/environments/$environment.yaml";
    my $env_config = -f $env_file ? LoadFile($env_file) : {};

    # Merge configs
    return $self->merge_configs($base, $env_config);
}

sub generate_docker_compose {
    my ($self, $config) = @_;

    my $template = <<'END_TEMPLATE';
version: '3.8'

services:
[% FOREACH service IN services %]
  [% service.name %]:
    image: [% service.image %]:[% version %]
    container_name: [% environment %]_[% service.name %]
    ports:
[% FOREACH port IN service.ports %]
      - "[% port %]"
[% END %]
    environment:
[% FOREACH env IN service.environment %]
      - [% env.key %]=[% env.value %]
[% END %]
    volumes:
[% FOREACH volume IN service.volumes %]
      - [% volume %]
[% END %]
[% IF service.depends_on %]
    depends_on:
[% FOREACH dep IN service.depends_on %]
      - [% dep %]
[% END %]
[% END %]
    restart: unless-stopped
    networks:
      - [% environment %]_network

[% END %]

networks:
  [% environment %]_network:
    driver: bridge

volumes:
[% FOREACH volume IN volumes %]
  [% volume %]:
[% END %]
END_TEMPLATE

    my $output;
    $self->{tt}->process(\$template, $config, \$output)
        or die $self->{tt}->error;

    path("docker-compose.$config->{environment}.yml")->spew($output);
}

sub generate_env_file {
    my ($self, $config) = @_;

    my @lines;
    push @lines, "# Environment: $config->{environment}";
    push @lines, "# Version: $config->{version}";
    push @lines, "";

    for my $key (sort keys %{$config->{env_vars}}) {
        my $value = $config->{env_vars}{$key};
        push @lines, "$key=$value";
    }

    path(".env.$config->{environment}")->spew(join("\n", @lines));
}

sub generate_nginx_config {
    my ($self, $config) = @_;

    my $template = <<'END_TEMPLATE';
upstream [% app_name %]_backend {
[% FOREACH server IN backend_servers %]
    server [% server %];
[% END %]
}

server {
    listen 80;
    server_name [% domain %];

    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name [% domain %];

    ssl_certificate /etc/letsencrypt/live/[% domain %]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[% domain %]/privkey.pem;

    location / {
        proxy_pass http://[% app_name %]_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/[% app_name %]/static/;
        expires 30d;
    }
}
END_TEMPLATE

    my $output;
    $self->{tt}->process(\$template, $config, \$output)
        or die $self->{tt}->error;

    path("nginx.$config->{environment}.conf")->spew($output);
}

sub generate_deploy_script {
    my ($self, $config) = @_;

    my $script = <<'END_SCRIPT';
#!/bin/bash
set -e

echo "Deploying [% app_name %] v[% version %] to [% environment %]"

# Pull latest images
docker-compose -f docker-compose.[% environment %].yml pull

# Stop current containers
docker-compose -f docker-compose.[% environment %].yml down

# Start new containers
docker-compose -f docker-compose.[% environment %].yml up -d

# Run migrations
docker-compose -f docker-compose.[% environment %].yml exec app ./migrate.sh

# Health check
sleep 10
curl -f http://localhost:[% health_check_port %]/health || exit 1

echo "Deployment completed successfully"
END_SCRIPT

    my $output;
    $self->{tt}->process(\$script, $config, \$output)
        or die $self->{tt}->error;

    my $file = path("deploy-$config->{environment}.sh");
    $file->spew($output);
    $file->chmod(0755);
}

sub merge_configs {
    my ($self, $base, $override) = @_;

    # Deep merge implementation
    my %merged = %$base;
    for my $key (keys %$override) {
        if (ref $override->{$key} eq 'HASH' && ref $merged{$key} eq 'HASH') {
            $merged{$key} = $self->merge_configs($merged{$key}, $override->{$key});
        } else {
            $merged{$key} = $override->{$key};
        }
    }

    return \%merged;
}

package main;

my ($environment, $version, $config_dir);
GetOptions(
    'environment=s' => \$environment,
    'version=s' => \$version,
    'config-dir=s' => \$config_dir,
) or die "Usage: $0 --environment=ENV --version=VERSION [--config-dir=DIR]\n";

$config_dir //= './deployment';

my $deployer = DeploymentManager->new($config_dir);
$deployer->deploy($environment, $version);
```

## Best Practices

1. **Separate configuration from code** - Use external files
2. **Support multiple environments** - Dev, staging, production
3. **Validate configurations** - Catch errors early
4. **Use templates for repetitive configs** - DRY principle
5. **Version your configurations** - Track changes
6. **Secure sensitive data** - Use environment variables or vaults
7. **Document configuration options** - Include examples
8. **Provide sensible defaults** - But allow overrides
9. **Cache processed templates** - Improve performance
10. **Test template output** - Ensure correctness

## Conclusion

Configuration management and templating turn rigid applications into flexible systems. Whether you're managing application settings across environments or generating complex documents, Perl's templating and configuration tools provide the flexibility and power you need.

Remember: configuration is code. Treat it with the same care—version it, test it, and document it.

---

*Next: CPAN - The Comprehensive Perl Archive Network. We'll explore how to find, use, and contribute to the treasure trove of Perl modules that makes Perl development so productive.*