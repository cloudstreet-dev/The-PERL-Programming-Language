# Chapter 11: Network Programming and Web Scraping

> "The Internet? We are not interested in it." - Bill Gates, 1993
> "Hold my beer." - Perl programmers, 1993

Perl and networking go together like coffee and late-night coding sessions. From the early days of CGI to modern REST APIs, Perl has been connecting systems and scraping data since before "web scraping" was even a term. This chapter covers everything from raw sockets to modern web automation.

## Socket Programming Basics

### TCP Client and Server

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use IO::Socket::INET;

# Simple TCP client
sub tcp_client {
    my ($host, $port) = @_;

    my $socket = IO::Socket::INET->new(
        PeerAddr => $host,
        PeerPort => $port,
        Proto    => 'tcp',
        Timeout  => 10,
    ) or die "Can't connect to $host:$port: $!";

    say "Connected to $host:$port";

    # Send data
    print $socket "GET / HTTP/1.0\r\n\r\n";

    # Read response
    while (my $line = <$socket>) {
        print $line;
    }

    close $socket;
}

# Simple TCP server
sub tcp_server {
    my ($port) = @_;

    my $server = IO::Socket::INET->new(
        LocalPort => $port,
        Proto     => 'tcp',
        Listen    => 5,
        Reuse     => 1,
    ) or die "Can't create server on port $port: $!";

    say "Server listening on port $port";

    while (my $client = $server->accept()) {
        my $peer = $client->peerhost();
        my $peerport = $client->peerport();
        say "Connection from $peer:$peerport";

        # Handle client
        print $client "HTTP/1.0 200 OK\r\n";
        print $client "Content-Type: text/plain\r\n\r\n";
        print $client "Hello from Perl server!\n";
        print $client "Your IP: $peer\n";
        print $client "Server time: " . localtime() . "\n";

        close $client;
    }
}

# Multi-threaded server
use threads;
use threads::shared;

sub threaded_server {
    my ($port) = @_;
    my $connections :shared = 0;

    my $server = IO::Socket::INET->new(
        LocalPort => $port,
        Proto     => 'tcp',
        Listen    => 10,
        Reuse     => 1,
    ) or die "Can't create server: $!";

    say "Threaded server on port $port";

    while (my $client = $server->accept()) {
        threads->create(sub {
            $connections++;
            handle_client($client, $connections);
            $connections--;
        })->detach();
    }
}

sub handle_client {
    my ($client, $id) = @_;

    say "[$id] New connection";

    while (my $line = <$client>) {
        chomp $line;
        last if $line eq 'quit';

        # Echo back
        print $client "You said: $line\n";
    }

    close $client;
    say "[$id] Connection closed";
}
```

### UDP Communication

```perl
# UDP client
sub udp_client {
    my ($host, $port, $message) = @_;

    my $socket = IO::Socket::INET->new(
        PeerAddr => $host,
        PeerPort => $port,
        Proto    => 'udp',
    ) or die "Can't create UDP socket: $!";

    $socket->send($message) or die "Send failed: $!";

    # Wait for response (with timeout)
    my $response;
    eval {
        local $SIG{ALRM} = sub { die "timeout\n" };
        alarm(5);
        $socket->recv($response, 1024);
        alarm(0);
    };

    if ($@) {
        warn "No response: $@";
    } else {
        say "Received: $response";
    }

    close $socket;
}

# UDP server
sub udp_server {
    my ($port) = @_;

    my $socket = IO::Socket::INET->new(
        LocalPort => $port,
        Proto     => 'udp',
    ) or die "Can't create UDP server: $!";

    say "UDP server listening on port $port";

    while (1) {
        my $data;
        my $client = $socket->recv($data, 1024);

        my ($port, $ip) = sockaddr_in($socket->peername);
        my $peer = inet_ntoa($ip);

        say "Received from $peer:$port: $data";

        # Send response
        $socket->send("ACK: $data");
    }
}
```

## HTTP Clients

### Using LWP::UserAgent

```perl
use LWP::UserAgent;
use HTTP::Request;
use HTTP::Cookies;

# Basic GET request
sub simple_get {
    my ($url) = @_;

    my $ua = LWP::UserAgent->new(
        timeout => 30,
        agent => 'MyPerlBot/1.0',
    );

    my $response = $ua->get($url);

    if ($response->is_success) {
        return $response->decoded_content;
    } else {
        die "Failed: " . $response->status_line;
    }
}

# POST with form data
sub post_form {
    my ($url, $data) = @_;

    my $ua = LWP::UserAgent->new();

    my $response = $ua->post($url, $data);

    return $response->decoded_content if $response->is_success;
    die "POST failed: " . $response->status_line;
}

# Advanced HTTP client with cookies and headers
sub advanced_client {
    my $ua = LWP::UserAgent->new(
        timeout => 30,
        max_redirect => 3,
        agent => 'Mozilla/5.0 (compatible; MyBot/1.0)',
    );

    # Cookie jar
    my $cookies = HTTP::Cookies->new(
        file => 'cookies.txt',
        autosave => 1,
    );
    $ua->cookie_jar($cookies);

    # Custom headers
    $ua->default_header('Accept' => 'application/json');
    $ua->default_header('Accept-Language' => 'en-US,en;q=0.9');

    # Authentication
    $ua->credentials('example.com:443', 'Protected', 'user', 'pass');

    # Proxy
    $ua->proxy(['http', 'https'], 'http://proxy.example.com:8080/');

    return $ua;
}

# Download file with progress
sub download_file {
    my ($url, $filename) = @_;

    my $ua = LWP::UserAgent->new();
    my $expected_length;
    my $bytes_received = 0;

    open my $fh, '>', $filename or die "Can't open $filename: $!";
    binmode $fh;

    my $response = $ua->get($url,
        ':content_cb' => sub {
            my ($chunk, $response) = @_;

            $expected_length //= $response->header('Content-Length');
            $bytes_received += length($chunk);

            print $fh $chunk;

            # Progress
            if ($expected_length) {
                my $percent = int(($bytes_received / $expected_length) * 100);
                print "\rDownloading: $percent%";
            }
        }
    );

    close $fh;
    print "\n";

    die "Download failed: " . $response->status_line
        unless $response->is_success;

    say "Downloaded $filename ($bytes_received bytes)";
}
```

### Modern HTTP with Mojo::UserAgent

```perl
use Mojo::UserAgent;
use Mojo::Promise;

# Async/await style requests
sub modern_http {
    my $ua = Mojo::UserAgent->new;

    # Simple GET
    my $result = $ua->get('https://api.github.com/users/perl')->result;
    my $json = $result->json;
    say "Perl has $json->{public_repos} public repos";

    # Concurrent requests
    my @urls = qw(
        https://perl.org
        https://metacpan.org
        https://perl.com
    );

    my @promises = map { $ua->get_p($_) } @urls;

    Mojo::Promise->all(@promises)->then(sub {
        my @results = @_;
        for my $tx (@results) {
            my $url = $tx->[0]->req->url;
            my $title = $tx->[0]->res->dom->at('title')->text;
            say "$url: $title";
        }
    })->wait;

    # WebSocket
    $ua->websocket('ws://echo.websocket.org' => sub {
        my ($ua, $tx) = @_;

        $tx->on(message => sub {
            my ($tx, $msg) = @_;
            say "Received: $msg";
            $tx->finish if $msg eq 'exit';
        });

        $tx->send('Hello, WebSocket!');
    });

    Mojo::IOLoop->start;
}
```

## Web Scraping

### HTML Parsing with Mojo::DOM

```perl
use Mojo::UserAgent;
use Mojo::DOM;

sub scrape_with_mojo {
    my ($url) = @_;

    my $ua = Mojo::UserAgent->new;
    my $dom = $ua->get($url)->result->dom;

    # CSS selectors
    my $title = $dom->at('title')->text;
    say "Title: $title";

    # Find all links
    $dom->find('a[href]')->each(sub {
        my $link = shift;
        say "Link: " . $link->attr('href') . " - " . $link->text;
    });

    # Extract table data
    my @rows;
    $dom->find('table tr')->each(sub {
        my $row = shift;
        my @cells = $row->find('td')->map('text')->each;
        push @rows, \@cells if @cells;
    });

    # Form extraction
    $dom->find('form')->each(sub {
        my $form = shift;
        say "Form action: " . $form->attr('action');

        $form->find('input')->each(sub {
            my $input = shift;
            say "  Input: " . ($input->attr('name') // 'unnamed');
        });
    });

    return \@rows;
}
```

### Web::Scraper for Structured Extraction

```perl
use Web::Scraper;
use URI;

# Define scraper
my $scraper = scraper {
    process 'title', 'title' => 'TEXT';
    process 'meta[name="description"]', 'description' => '@content';
    process 'h1', 'heading[]' => 'TEXT';
    process 'a', 'links[]' => {
        text => 'TEXT',
        href => '@href',
    };
    process 'img', 'images[]' => '@src';
};

# Use scraper
my $url = URI->new('https://example.com');
my $result = $scraper->scrape($url);

use Data::Dumper;
print Dumper($result);

# Complex scraping patterns
sub scrape_products {
    my ($url) = @_;

    my $products = scraper {
        process '.product', 'products[]' => scraper {
            process '.name', 'name' => 'TEXT';
            process '.price', 'price' => 'TEXT';
            process '.description', 'desc' => 'TEXT';
            process 'img', 'image' => '@src';
            process 'a.details', 'link' => '@href';
        };
        process '.pagination .next', 'next_page' => '@href';
    };

    my @all_products;
    my $current_url = $url;

    while ($current_url) {
        my $result = $products->scrape(URI->new($current_url));
        push @all_products, @{$result->{products} || []};

        # Follow pagination
        $current_url = $result->{next_page};
        last if @all_products > 100;  # Limit
    }

    return \@all_products;
}
```

### JavaScript-Heavy Sites with Selenium

```perl
use Selenium::Chrome;
use Selenium::Waiter qw(wait_until);

sub scrape_dynamic_site {
    my ($url) = @_;

    # Start Chrome
    my $driver = Selenium::Chrome->new(
        extra_capabilities => {
            'goog:chromeOptions' => {
                args => ['--headless', '--disable-gpu'],
            }
        }
    );

    $driver->get($url);

    # Wait for dynamic content
    wait_until { $driver->find_element('.products-loaded') };

    # Scroll to load more content
    $driver->execute_script('window.scrollTo(0, document.body.scrollHeight)');
    sleep(2);

    # Extract data after JS rendering
    my @products;
    my $elements = $driver->find_elements('.product-card');

    for my $element (@$elements) {
        push @products, {
            name => $element->find_child_element('.name')->get_text(),
            price => $element->find_child_element('.price')->get_text(),
        };
    }

    # Interact with page
    my $button = $driver->find_element('#load-more');
    $button->click() if $button;

    $driver->quit();

    return \@products;
}
```

## REST API Clients

### Building a REST Client

```perl
package RESTClient;
use Modern::Perl '2023';
use LWP::UserAgent;
use JSON::XS;
use URI::Escape;

sub new {
    my ($class, $base_url, $options) = @_;

    my $self = {
        base_url => $base_url,
        ua => LWP::UserAgent->new(
            timeout => $options->{timeout} // 30,
            agent => $options->{agent} // 'RESTClient/1.0',
        ),
        headers => $options->{headers} // {},
        auth => $options->{auth},
    };

    return bless $self, $class;
}

sub request {
    my ($self, $method, $path, $data) = @_;

    my $url = $self->{base_url} . $path;
    my $req = HTTP::Request->new($method => $url);

    # Add headers
    for my $header (keys %{$self->{headers}}) {
        $req->header($header => $self->{headers}{$header});
    }

    # Add auth
    if ($self->{auth}) {
        if ($self->{auth}{type} eq 'basic') {
            $req->authorization_basic(
                $self->{auth}{username},
                $self->{auth}{password}
            );
        } elsif ($self->{auth}{type} eq 'bearer') {
            $req->header(Authorization => "Bearer $self->{auth}{token}");
        }
    }

    # Add data
    if ($data) {
        if ($method eq 'GET') {
            # Add as query params
            my @params;
            for my $key (keys %$data) {
                push @params, uri_escape($key) . '=' . uri_escape($data->{$key});
            }
            $url .= '?' . join('&', @params);
            $req->uri($url);
        } else {
            # Send as JSON body
            $req->header('Content-Type' => 'application/json');
            $req->content(encode_json($data));
        }
    }

    my $response = $self->{ua}->request($req);

    if ($response->is_success) {
        my $content = $response->decoded_content;
        return $response->header('Content-Type') =~ /json/
            ? decode_json($content)
            : $content;
    } else {
        die "Request failed: " . $response->status_line;
    }
}

sub get    { shift->request('GET', @_) }
sub post   { shift->request('POST', @_) }
sub put    { shift->request('PUT', @_) }
sub delete { shift->request('DELETE', @_) }
sub patch  { shift->request('PATCH', @_) }

# Usage
package main;

my $api = RESTClient->new('https://api.example.com', {
    headers => {
        'Accept' => 'application/json',
        'X-API-Version' => '2.0',
    },
    auth => {
        type => 'bearer',
        token => 'your-api-token',
    },
});

# GET /users
my $users = $api->get('/users', { limit => 10, offset => 0 });

# POST /users
my $new_user = $api->post('/users', {
    name => 'Alice',
    email => 'alice@example.com',
});

# PUT /users/123
$api->put('/users/123', { name => 'Alice Smith' });

# DELETE /users/123
$api->delete('/users/123');
```

## Email and SMTP

### Sending Email

```perl
use Email::Simple;
use Email::Sender::Simple qw(sendmail);
use Email::Sender::Transport::SMTP;
use Email::MIME;

# Simple email
sub send_simple_email {
    my ($to, $subject, $body) = @_;

    my $email = Email::Simple->create(
        header => [
            To      => $to,
            From    => 'sender@example.com',
            Subject => $subject,
        ],
        body => $body,
    );

    sendmail($email);
}

# Email with attachments
sub send_email_with_attachment {
    my ($to, $subject, $body, $attachment) = @_;

    my @parts = (
        Email::MIME->create(
            attributes => {
                content_type => 'text/plain',
                charset      => 'UTF-8',
            },
            body => $body,
        ),
    );

    if ($attachment && -f $attachment) {
        open my $fh, '<:raw', $attachment or die $!;
        my $data = do { local $/; <$fh> };
        close $fh;

        push @parts, Email::MIME->create(
            attributes => {
                filename     => basename($attachment),
                content_type => 'application/octet-stream',
                encoding     => 'base64',
                disposition  => 'attachment',
            },
            body => $data,
        );
    }

    my $email = Email::MIME->create(
        header => [
            To      => $to,
            From    => 'sender@example.com',
            Subject => $subject,
        ],
        parts => \@parts,
    );

    # Custom SMTP transport
    my $transport = Email::Sender::Transport::SMTP->new({
        host => 'smtp.gmail.com',
        port => 587,
        sasl_username => 'your@gmail.com',
        sasl_password => 'your-password',
        ssl => 'starttls',
    });

    sendmail($email, { transport => $transport });
}

sub basename { (split /\//, $_[0])[-1] }
```

## Real-World Example: Web Monitor

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use LWP::UserAgent;
use Digest::MD5 qw(md5_hex);
use Time::HiRes qw(time);
use Email::Simple;
use Email::Sender::Simple qw(sendmail);

package WebMonitor;

sub new {
    my ($class, $config) = @_;

    return bless {
        config => $config,
        ua => LWP::UserAgent->new(
            timeout => 30,
            agent => 'WebMonitor/1.0',
        ),
        state => {},
    }, $class;
}

sub monitor {
    my ($self) = @_;

    while (1) {
        for my $site (@{$self->{config}{sites}}) {
            $self->check_site($site);
        }

        sleep($self->{config}{interval} // 300);  # 5 minutes default
    }
}

sub check_site {
    my ($self, $site) = @_;

    my $start_time = time();
    my $response = $self->{ua}->get($site->{url});
    my $response_time = time() - $start_time;

    my $status = {
        url => $site->{url},
        timestamp => time(),
        response_time => $response_time,
        status_code => $response->code,
        success => $response->is_success,
    };

    # Check for changes
    if ($response->is_success) {
        my $content = $response->decoded_content;
        my $hash = md5_hex($content);

        my $prev_hash = $self->{state}{$site->{url}}{hash} // '';
        if ($prev_hash && $hash ne $prev_hash) {
            $status->{changed} = 1;
            $self->alert("Content changed", $site, $status);
        }

        $self->{state}{$site->{url}}{hash} = $hash;

        # Check for keywords
        if ($site->{keywords}) {
            for my $keyword (@{$site->{keywords}}) {
                if ($content =~ /$keyword/i) {
                    $self->alert("Keyword found: $keyword", $site, $status);
                }
            }
        }

        # Check response time
        if ($site->{max_response_time} &&
            $response_time > $site->{max_response_time}) {
            $self->alert("Slow response", $site, $status);
        }
    } else {
        # Site is down
        $self->alert("Site down", $site, $status);
    }

    $self->log_status($status);
}

sub alert {
    my ($self, $reason, $site, $status) = @_;

    say "[ALERT] $reason for $site->{url}";

    if ($self->{config}{email_alerts}) {
        my $email = Email::Simple->create(
            header => [
                To      => $self->{config}{alert_email},
                From    => 'monitor@example.com',
                Subject => "Web Monitor Alert: $reason",
            ],
            body => $self->format_alert($reason, $site, $status),
        );

        eval { sendmail($email) };
        warn "Failed to send alert: $@" if $@;
    }
}

sub format_alert {
    my ($self, $reason, $site, $status) = @_;

    return <<EOF;
Alert: $reason

Site: $site->{url}
Time: @{[scalar localtime($status->{timestamp})]}
Status Code: $status->{status_code}
Response Time: @{[sprintf "%.2f", $status->{response_time}]} seconds

Please investigate.
EOF
}

sub log_status {
    my ($self, $status) = @_;

    open my $fh, '>>', 'monitor.log' or die $!;
    print $fh join("\t",
        $status->{timestamp},
        $status->{url},
        $status->{status_code},
        $status->{response_time},
        $status->{success} ? 'OK' : 'FAIL'
    ), "\n";
    close $fh;
}

package main;

my $config = {
    interval => 60,  # Check every minute
    email_alerts => 1,
    alert_email => 'admin@example.com',
    sites => [
        {
            url => 'https://example.com',
            max_response_time => 5,
            keywords => ['error', 'maintenance'],
        },
        {
            url => 'https://api.example.com/health',
            max_response_time => 2,
        },
    ],
};

my $monitor = WebMonitor->new($config);
$monitor->monitor();
```

## Best Practices

1. **Set appropriate timeouts** - Network calls can hang forever
2. **Handle redirects carefully** - Limit redirect chains
3. **Respect robots.txt** - Be a good web citizen
4. **Add delays between requests** - Don't hammer servers
5. **Use connection pooling** - Reuse connections when possible
6. **Implement retry logic** - Networks are unreliable
7. **Cache responses** - Reduce unnecessary requests
8. **Set a proper User-Agent** - Identify your bot
9. **Handle encodings properly** - UTF-8 isn't universal
10. **Log failures for debugging** - Network issues are common

## Conclusion

Network programming and web scraping in Perl combine power with practicality. Whether you're building robust API clients, scraping data from websites, or creating network services, Perl provides the tools to do it efficiently. The key is choosing the right tool for each task and respecting the services you interact with.

Remember: just because you can scrape something doesn't mean you should. Always check terms of service and robots.txt, and be respectful of the resources you're accessing.

---

*Next: Database operations with DBI. We'll explore how Perl interfaces with databases, from simple queries to complex transactions and migrations.*