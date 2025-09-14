# Chapter 21: RESTful APIs and Web Services

*"In the interconnected world of modern software, Perl serves as both a capable API consumer and a powerful service provider."*

## The Web Services Landscape

In 2025, RESTful APIs remain the lingua franca of web services. While GraphQL and gRPC have their niches, REST's simplicity and ubiquity make it essential for any systems programmer. Perl, with its excellent HTTP libraries and JSON handling, excels at both consuming and providing web services.

## Building a RESTful API Server

Let's start by building a complete REST API using Mojolicious, Perl's most modern web framework:

```perl
#!/usr/bin/env perl
use Mojolicious::Lite -signatures;
use Mojo::JSON qw(encode_json decode_json);
use DBI;
use Try::Tiny;

# Initialize database
my $dbh = DBI->connect("dbi:SQLite:dbname=api.db","","", {
    RaiseError => 1,
    PrintError => 0,
    AutoCommit => 1,
});

# Create tables if needed
$dbh->do(q{
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        email TEXT UNIQUE NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
});

$dbh->do(q{
    CREATE TABLE IF NOT EXISTS posts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER NOT NULL,
        title TEXT NOT NULL,
        content TEXT,
        published BOOLEAN DEFAULT 0,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users(id)
    )
});

# Middleware for JSON error handling
hook before_dispatch => sub ($c) {
    $c->res->headers->header('Content-Type' => 'application/json');
};

hook after_dispatch => sub ($c) {
    if ($c->res->code >= 400 && !$c->res->body) {
        $c->render(json => {
            error => $c->res->message || 'An error occurred',
            status => $c->res->code
        });
    }
};

# Helper for database operations
helper db => sub { $dbh };

# Helper for pagination
helper paginate => sub ($c, $query, $count_query, @params) {
    my $page = $c->param('page') // 1;
    my $limit = $c->param('limit') // 10;
    my $offset = ($page - 1) * $limit;

    # Get total count
    my ($total) = $dbh->selectrow_array($count_query, {}, @params);

    # Get paginated results
    my $results = $dbh->selectall_arrayref(
        "$query LIMIT ? OFFSET ?",
        { Slice => {} },
        @params, $limit, $offset
    );

    return {
        data => $results,
        meta => {
            page => $page,
            limit => $limit,
            total => $total,
            pages => int(($total + $limit - 1) / $limit)
        }
    };
};

# Routes

# Health check endpoint
get '/health' => sub ($c) {
    $c->render(json => {
        status => 'healthy',
        timestamp => time(),
        version => '1.0.0'
    });
};

# User endpoints
group {
    under '/api/v1';

    # List users with pagination
    get '/users' => sub ($c) {
        my $result = $c->paginate(
            "SELECT * FROM users ORDER BY created_at DESC",
            "SELECT COUNT(*) FROM users"
        );
        $c->render(json => $result);
    };

    # Get single user
    get '/users/:id' => sub ($c) {
        my $id = $c->param('id');

        my $user = $dbh->selectrow_hashref(
            "SELECT * FROM users WHERE id = ?",
            {}, $id
        );

        return $c->render(json => { error => 'User not found' }, status => 404)
            unless $user;

        # Get user's posts
        $user->{posts} = $dbh->selectall_arrayref(
            "SELECT id, title, published FROM posts WHERE user_id = ?",
            { Slice => {} }, $id
        );

        $c->render(json => { data => $user });
    };

    # Create user
    post '/users' => sub ($c) {
        my $json = $c->req->json;

        # Validation
        my @required = qw(username email);
        for my $field (@required) {
            return $c->render(
                json => { error => "Missing required field: $field" },
                status => 400
            ) unless $json->{$field};
        }

        # Check for duplicates
        my $existing = $dbh->selectrow_hashref(
            "SELECT id FROM users WHERE username = ? OR email = ?",
            {}, $json->{username}, $json->{email}
        );

        return $c->render(
            json => { error => 'Username or email already exists' },
            status => 409
        ) if $existing;

        # Insert user
        try {
            $dbh->do(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                {}, $json->{username}, $json->{email}
            );

            my $id = $dbh->last_insert_id(undef, undef, 'users', undef);
            my $user = $dbh->selectrow_hashref(
                "SELECT * FROM users WHERE id = ?",
                {}, $id
            );

            $c->render(json => { data => $user }, status => 201);
        }
        catch {
            $c->render(json => { error => "Failed to create user: $_" }, status => 500);
        };
    };

    # Update user
    put '/users/:id' => sub ($c) {
        my $id = $c->param('id');
        my $json = $c->req->json;

        # Check if user exists
        my $user = $dbh->selectrow_hashref(
            "SELECT * FROM users WHERE id = ?",
            {}, $id
        );

        return $c->render(json => { error => 'User not found' }, status => 404)
            unless $user;

        # Build update query
        my @fields;
        my @values;

        for my $field (qw(username email)) {
            if (exists $json->{$field}) {
                push @fields, "$field = ?";
                push @values, $json->{$field};
            }
        }

        return $c->render(json => { error => 'No fields to update' }, status => 400)
            unless @fields;

        push @values, $id;

        try {
            $dbh->do(
                "UPDATE users SET " . join(', ', @fields) .
                ", updated_at = CURRENT_TIMESTAMP WHERE id = ?",
                {}, @values
            );

            $user = $dbh->selectrow_hashref(
                "SELECT * FROM users WHERE id = ?",
                {}, $id
            );

            $c->render(json => { data => $user });
        }
        catch {
            $c->render(json => { error => "Failed to update user: $_" }, status => 500);
        };
    };

    # Delete user
    del '/users/:id' => sub ($c) {
        my $id = $c->param('id');

        my $user = $dbh->selectrow_hashref(
            "SELECT * FROM users WHERE id = ?",
            {}, $id
        );

        return $c->render(json => { error => 'User not found' }, status => 404)
            unless $user;

        try {
            $dbh->begin_work;

            # Delete user's posts first
            $dbh->do("DELETE FROM posts WHERE user_id = ?", {}, $id);

            # Delete user
            $dbh->do("DELETE FROM users WHERE id = ?", {}, $id);

            $dbh->commit;

            $c->render(json => { message => 'User deleted successfully' });
        }
        catch {
            $dbh->rollback;
            $c->render(json => { error => "Failed to delete user: $_" }, status => 500);
        };
    };

    # Post endpoints

    # List all posts
    get '/posts' => sub ($c) {
        my $published_only = $c->param('published') // 0;

        my $where = $published_only ? "WHERE published = 1" : "";

        my $result = $c->paginate(
            "SELECT p.*, u.username
             FROM posts p
             JOIN users u ON p.user_id = u.id
             $where
             ORDER BY p.created_at DESC",
            "SELECT COUNT(*) FROM posts p $where"
        );

        $c->render(json => $result);
    };

    # Create post
    post '/posts' => sub ($c) {
        my $json = $c->req->json;

        # Validation
        my @required = qw(user_id title content);
        for my $field (@required) {
            return $c->render(
                json => { error => "Missing required field: $field" },
                status => 400
            ) unless defined $json->{$field};
        }

        # Check if user exists
        my $user = $dbh->selectrow_hashref(
            "SELECT id FROM users WHERE id = ?",
            {}, $json->{user_id}
        );

        return $c->render(json => { error => 'User not found' }, status => 404)
            unless $user;

        try {
            $dbh->do(
                "INSERT INTO posts (user_id, title, content, published) VALUES (?, ?, ?, ?)",
                {}, $json->{user_id}, $json->{title}, $json->{content}, $json->{published} // 0
            );

            my $id = $dbh->last_insert_id(undef, undef, 'posts', undef);
            my $post = $dbh->selectrow_hashref(
                "SELECT p.*, u.username
                 FROM posts p
                 JOIN users u ON p.user_id = u.id
                 WHERE p.id = ?",
                {}, $id
            );

            $c->render(json => { data => $post }, status => 201);
        }
        catch {
            $c->render(json => { error => "Failed to create post: $_" }, status => 500);
        };
    };

    # Search posts
    get '/posts/search' => sub ($c) {
        my $query = $c->param('q');

        return $c->render(json => { error => 'Query parameter required' }, status => 400)
            unless $query;

        my $posts = $dbh->selectall_arrayref(
            "SELECT p.*, u.username
             FROM posts p
             JOIN users u ON p.user_id = u.id
             WHERE p.title LIKE ? OR p.content LIKE ?
             ORDER BY p.created_at DESC
             LIMIT 20",
            { Slice => {} },
            "%$query%", "%$query%"
        );

        $c->render(json => { data => $posts });
    };
};

# Start the application
app->start;
```

## Consuming External APIs

Now let's build a comprehensive API client that demonstrates best practices:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package API::Client {
    use Moo;
    use Types::Standard qw(Str Int HashRef Bool);
    use LWP::UserAgent;
    use HTTP::Request;
    use JSON::XS;
    use URI;
    use Try::Tiny;
    use Time::HiRes qw(time);
    use Cache::LRU;
    use Log::Any qw($log);

    has base_url => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    has api_key => (
        is => 'ro',
        isa => Str,
        predicate => 'has_api_key',
    );

    has timeout => (
        is => 'ro',
        isa => Int,
        default => 30,
    );

    has retry_count => (
        is => 'ro',
        isa => Int,
        default => 3,
    );

    has retry_delay => (
        is => 'ro',
        isa => Int,
        default => 1,
    );

    has cache_ttl => (
        is => 'ro',
        isa => Int,
        default => 300,  # 5 minutes
    );

    has user_agent => (
        is => 'lazy',
        isa => sub { ref $_[0] eq 'LWP::UserAgent' },
    );

    has json => (
        is => 'lazy',
        default => sub { JSON::XS->new->utf8->pretty->canonical },
    );

    has cache => (
        is => 'lazy',
        default => sub { Cache::LRU->new(size => 100) },
    );

    has _cache_timestamps => (
        is => 'ro',
        default => sub { {} },
    );

    sub _build_user_agent ($self) {
        my $ua = LWP::UserAgent->new(
            timeout => $self->timeout,
            agent => 'Perl API Client/1.0',
        );

        # Add default headers
        $ua->default_header('Accept' => 'application/json');
        $ua->default_header('Content-Type' => 'application/json');

        # Add API key if provided
        if ($self->has_api_key) {
            $ua->default_header('Authorization' => 'Bearer ' . $self->api_key);
        }

        return $ua;
    }

    sub request ($self, $method, $endpoint, $params = {}, $body = undef) {
        # Build full URL
        my $uri = URI->new($self->base_url . $endpoint);

        # Add query parameters for GET requests
        if ($method eq 'GET' && $params && %$params) {
            $uri->query_form($params);
        }

        # Check cache for GET requests
        my $cache_key = "$method:$uri";
        if ($method eq 'GET') {
            my $cached = $self->_get_cached($cache_key);
            return $cached if $cached;
        }

        # Prepare request
        my $request = HTTP::Request->new($method => $uri);

        # Add body for POST/PUT/PATCH requests
        if ($body && ($method eq 'POST' || $method eq 'PUT' || $method eq 'PATCH')) {
            my $json_body = ref $body ? $self->json->encode($body) : $body;
            $request->content($json_body);
        }

        # Execute request with retries
        my $response;
        my $attempt = 0;
        my $last_error;

        while ($attempt < $self->retry_count) {
            $attempt++;

            $log->debug("API request attempt $attempt: $method $uri");
            my $start_time = time();

            try {
                $response = $self->user_agent->request($request);

                my $elapsed = sprintf("%.3f", time() - $start_time);
                $log->debug("API response: " . $response->code . " (${elapsed}s)");

                # Check for rate limiting
                if ($response->code == 429) {
                    my $retry_after = $response->header('Retry-After') || $self->retry_delay * $attempt;
                    $log->warn("Rate limited, waiting ${retry_after}s");
                    sleep($retry_after);
                    next;
                }

                # Success or client error (no retry needed)
                if ($response->is_success || ($response->code >= 400 && $response->code < 500)) {
                    last;
                }

                # Server error - retry
                if ($response->code >= 500) {
                    $last_error = "Server error: " . $response->status_line;
                    $log->warn($last_error);
                    sleep($self->retry_delay * $attempt) if $attempt < $self->retry_count;
                    next;
                }
            }
            catch {
                $last_error = "Request failed: $_";
                $log->error($last_error);
                sleep($self->retry_delay * $attempt) if $attempt < $self->retry_count;
            };
        }

        # Check final response
        unless ($response) {
            die "API request failed after $attempt attempts: $last_error";
        }

        unless ($response->is_success) {
            my $error = $response->status_line;
            if ($response->content) {
                try {
                    my $error_data = $self->json->decode($response->content);
                    $error = $error_data->{error} || $error_data->{message} || $error;
                }
                catch {
                    # Use status line if JSON decode fails
                };
            }
            die "API error: $error";
        }

        # Parse response
        my $data;
        if ($response->content) {
            try {
                $data = $self->json->decode($response->content);
            }
            catch {
                die "Failed to parse API response: $_";
            };
        }

        # Cache successful GET requests
        if ($method eq 'GET' && $data) {
            $self->_set_cached($cache_key, $data);
        }

        return $data;
    }

    sub _get_cached ($self, $key) {
        my $timestamp = $self->_cache_timestamps->{$key};
        return unless $timestamp;

        if (time() - $timestamp > $self->cache_ttl) {
            delete $self->_cache_timestamps->{$key};
            $self->cache->remove($key);
            return;
        }

        return $self->cache->get($key);
    }

    sub _set_cached ($self, $key, $value) {
        $self->cache->set($key => $value);
        $self->_cache_timestamps->{$key} = time();
    }

    # Convenience methods
    sub get ($self, $endpoint, $params = {}) {
        return $self->request('GET', $endpoint, $params);
    }

    sub post ($self, $endpoint, $body = {}, $params = {}) {
        return $self->request('POST', $endpoint, $params, $body);
    }

    sub put ($self, $endpoint, $body = {}, $params = {}) {
        return $self->request('PUT', $endpoint, $params, $body);
    }

    sub patch ($self, $endpoint, $body = {}, $params = {}) {
        return $self->request('PATCH', $endpoint, $params, $body);
    }

    sub delete ($self, $endpoint, $params = {}) {
        return $self->request('DELETE', $endpoint, $params);
    }
}

# Example: GitHub API client
package GitHub::Client {
    use Moo;
    use Types::Standard qw(Str);
    extends 'API::Client';

    has '+base_url' => (
        default => 'https://api.github.com',
    );

    has username => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    sub user_repos ($self) {
        return $self->get("/users/" . $self->username . "/repos", {
            sort => 'updated',
            per_page => 100,
        });
    }

    sub repo_info ($self, $repo) {
        return $self->get("/repos/" . $self->username . "/$repo");
    }

    sub repo_issues ($self, $repo, $state = 'open') {
        return $self->get("/repos/" . $self->username . "/$repo/issues", {
            state => $state,
            per_page => 100,
        });
    }

    sub create_issue ($self, $repo, $title, $body, $labels = []) {
        return $self->post("/repos/" . $self->username . "/$repo/issues", {
            title => $title,
            body => $body,
            labels => $labels,
        });
    }
}

# Usage example
use Log::Any::Adapter 'Stdout';

my $github = GitHub::Client->new(
    username => 'torvalds',
    api_key => $ENV{GITHUB_TOKEN},  # Optional
);

try {
    # Get user's repositories
    my $repos = $github->user_repos();

    say "Found " . scalar(@$repos) . " repositories";

    for my $repo (@$repos) {
        printf("%-30s ⭐ %-6d 🍴 %-6d %s\n",
            $repo->{name},
            $repo->{stargazers_count},
            $repo->{forks_count},
            $repo->{description} // ''
        );
    }

    # Get specific repo details
    my $linux = $github->repo_info('linux');
    say "\nLinux kernel repo:";
    say "  Stars: " . $linux->{stargazers_count};
    say "  Forks: " . $linux->{forks_count};
    say "  Open Issues: " . $linux->{open_issues_count};
}
catch {
    warn "Error: $_";
};
```

## WebSocket Communication

For real-time applications, WebSocket support is essential:

```perl
#!/usr/bin/env perl
use Mojolicious::Lite -signatures;
use Mojo::JSON qw(encode_json decode_json);

# Store active connections
my %clients;
my $client_id = 0;

# WebSocket chat server
websocket '/ws' => sub ($c) {
    my $id = ++$client_id;

    # Store connection
    $clients{$id} = $c;

    # Send welcome message
    $c->send(encode_json({
        type => 'system',
        message => "Welcome! You are client #$id",
        timestamp => time(),
    }));

    # Notify others of new connection
    broadcast({
        type => 'system',
        message => "Client #$id joined",
        timestamp => time(),
    }, $id);

    # Handle incoming messages
    $c->on(message => sub ($c, $msg) {
        my $data;

        # Try to parse JSON
        eval { $data = decode_json($msg) };
        if ($@) {
            $c->send(encode_json({
                type => 'error',
                message => 'Invalid JSON',
            }));
            return;
        }

        # Handle different message types
        if ($data->{type} eq 'chat') {
            broadcast({
                type => 'chat',
                from => "Client #$id",
                message => $data->{message},
                timestamp => time(),
            });
        }
        elsif ($data->{type} eq 'ping') {
            $c->send(encode_json({
                type => 'pong',
                timestamp => time(),
            }));
        }
    });

    # Handle disconnection
    $c->on(finish => sub ($c, $code, $reason) {
        delete $clients{$id};
        broadcast({
            type => 'system',
            message => "Client #$id disconnected",
            timestamp => time(),
        });
    });
};

sub broadcast ($message, $exclude_id = undef) {
    my $json = encode_json($message);

    for my $id (keys %clients) {
        next if defined $exclude_id && $id == $exclude_id;
        $clients{$id}->send($json);
    }
}

# Serve HTML client
get '/' => sub ($c) {
    $c->render(inline => <<'HTML');
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Chat</title>
    <style>
        #messages { height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; }
        .system { color: #666; font-style: italic; }
        .chat { margin: 5px 0; }
        .timestamp { color: #999; font-size: 0.8em; }
    </style>
</head>
<body>
    <h1>WebSocket Chat</h1>
    <div id="messages"></div>
    <input type="text" id="input" placeholder="Type a message..." style="width: 300px;">
    <button onclick="sendMessage()">Send</button>

    <script>
        const ws = new WebSocket('ws://localhost:3000/ws');
        const messages = document.getElementById('messages');
        const input = document.getElementById('input');

        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            const div = document.createElement('div');
            const time = new Date(data.timestamp * 1000).toLocaleTimeString();

            if (data.type === 'system') {
                div.className = 'system';
                div.innerHTML = `<span class="timestamp">[${time}]</span> ${data.message}`;
            } else if (data.type === 'chat') {
                div.className = 'chat';
                div.innerHTML = `<span class="timestamp">[${time}]</span> <b>${data.from}:</b> ${data.message}`;
            }

            messages.appendChild(div);
            messages.scrollTop = messages.scrollHeight;
        };

        function sendMessage() {
            if (input.value.trim()) {
                ws.send(JSON.stringify({
                    type: 'chat',
                    message: input.value
                }));
                input.value = '';
            }
        }

        input.addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });

        // Periodic ping to keep connection alive
        setInterval(() => {
            ws.send(JSON.stringify({ type: 'ping' }));
        }, 30000);
    </script>
</body>
</html>
HTML
};

app->start;
```

## GraphQL Integration

While REST dominates, GraphQL offers powerful query capabilities:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package GraphQL::Client {
    use Moo;
    use Types::Standard qw(Str HashRef);
    use LWP::UserAgent;
    use HTTP::Request;
    use JSON::XS;
    use Try::Tiny;

    has endpoint => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    has headers => (
        is => 'ro',
        isa => HashRef,
        default => sub { {} },
    );

    has ua => (
        is => 'lazy',
        default => sub {
            my $ua = LWP::UserAgent->new(timeout => 30);
            $ua->default_header('Content-Type' => 'application/json');
            return $ua;
        },
    );

    has json => (
        is => 'lazy',
        default => sub { JSON::XS->new->utf8->pretty },
    );

    sub query ($self, $query, $variables = {}, $operation_name = undef) {
        my $request_body = {
            query => $query,
            variables => $variables,
        };

        $request_body->{operationName} = $operation_name if $operation_name;

        my $request = HTTP::Request->new('POST' => $self->endpoint);
        $request->content($self->json->encode($request_body));

        # Add custom headers
        for my $header (keys %{$self->headers}) {
            $request->header($header => $self->headers->{$header});
        }

        my $response = $self->ua->request($request);

        unless ($response->is_success) {
            die "GraphQL request failed: " . $response->status_line;
        }

        my $data = try {
            $self->json->decode($response->content);
        }
        catch {
            die "Failed to parse GraphQL response: $_";
        };

        if ($data->{errors}) {
            my $errors = join(", ", map { $_->{message} } @{$data->{errors}});
            die "GraphQL errors: $errors";
        }

        return $data->{data};
    }

    sub introspect ($self) {
        my $introspection_query = q{
            query IntrospectionQuery {
                __schema {
                    types {
                        name
                        kind
                        description
                        fields {
                            name
                            type {
                                name
                                kind
                            }
                        }
                    }
                }
            }
        };

        return $self->query($introspection_query);
    }
}

# Example: GitHub GraphQL API
my $github_graphql = GraphQL::Client->new(
    endpoint => 'https://api.github.com/graphql',
    headers => {
        Authorization => "Bearer $ENV{GITHUB_TOKEN}",
    },
);

# Query repository information
my $repo_query = q{
    query GetRepository($owner: String!, $name: String!) {
        repository(owner: $owner, name: $name) {
            name
            description
            stargazerCount
            forkCount
            primaryLanguage {
                name
            }
            issues(states: OPEN) {
                totalCount
            }
            pullRequests(states: OPEN) {
                totalCount
            }
            releases(last: 5) {
                nodes {
                    tagName
                    publishedAt
                }
            }
        }
    }
};

try {
    my $data = $github_graphql->query($repo_query, {
        owner => 'perl',
        name => 'perl5',
    });

    my $repo = $data->{repository};
    say "Repository: $repo->{name}";
    say "Description: $repo->{description}";
    say "Stars: $repo->{stargazerCount}";
    say "Forks: $repo->{forkCount}";
    say "Language: " . ($repo->{primaryLanguage}{name} // 'Unknown');
    say "Open Issues: " . $repo->{issues}{totalCount};
    say "Open PRs: " . $repo->{pullRequests}{totalCount};

    say "\nRecent Releases:";
    for my $release (@{$repo->{releases}{nodes}}) {
        say "  $release->{tagName} - $release->{publishedAt}";
    }
}
catch {
    warn "Error: $_";
};
```

## API Documentation with OpenAPI

Generate OpenAPI specifications for your Perl APIs:

```perl
#!/usr/bin/env perl
use Mojolicious::Lite -signatures;
use Mojolicious::Plugin::OpenAPI;

# Load OpenAPI specification
plugin OpenAPI => {
    url => "data:///openapi.yaml",
    route => app->routes->under("/api")->to("auth#check"),
};

# Controller actions
package MyApp::Controller::Pet;
use Mojo::Base 'Mojolicious::Controller', -signatures;

sub list ($c) {
    # Get query parameters with validation from OpenAPI spec
    my $input = $c->validation->output;

    my @pets = (
        { id => 1, name => "Fluffy", type => "cat" },
        { id => 2, name => "Rover", type => "dog" },
    );

    # Filter by type if specified
    if ($input->{type}) {
        @pets = grep { $_->{type} eq $input->{type} } @pets;
    }

    $c->render(openapi => \@pets);
}

sub add ($c) {
    my $input = $c->validation->output;

    # Simulate adding pet
    my $pet = {
        id => int(rand(1000)),
        %{$input->{body}},
    };

    $c->render(openapi => $pet, status => 201);
}

package MyApp::Controller::Auth;
use Mojo::Base 'Mojolicious::Controller', -signatures;

sub check ($c) {
    my $api_key = $c->req->headers->header('X-API-Key');

    unless ($api_key && $api_key eq 'secret123') {
        return $c->render(
            openapi => { error => 'Unauthorized' },
            status => 401
        );
    }

    return 1;
}

app->start;

__DATA__
@@ openapi.yaml
openapi: 3.0.0
info:
  title: Pet Store API
  version: 1.0.0
  description: A simple pet store API
servers:
  - url: /api
paths:
  /pets:
    get:
      operationId: listPets
      summary: List all pets
      x-mojo-to: "pet#list"
      parameters:
        - name: type
          in: query
          schema:
            type: string
            enum: [cat, dog, bird]
      responses:
        200:
          description: List of pets
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Pet'
    post:
      operationId: addPet
      summary: Add a new pet
      x-mojo-to: "pet#add"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/NewPet'
      responses:
        201:
          description: Pet created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Pet'
components:
  schemas:
    Pet:
      type: object
      required:
        - id
        - name
        - type
      properties:
        id:
          type: integer
        name:
          type: string
        type:
          type: string
    NewPet:
      type: object
      required:
        - name
        - type
      properties:
        name:
          type: string
        type:
          type: string
  securitySchemes:
    ApiKey:
      type: apiKey
      in: header
      name: X-API-Key
security:
  - ApiKey: []
```

## API Testing and Mocking

Comprehensive testing ensures reliable APIs:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Test::More;
use Test::Mojo;
use Test::Deep;
use Mojolicious::Lite;

# Define test API
get '/users' => sub {
    shift->render(json => [
        { id => 1, name => 'Alice' },
        { id => 2, name => 'Bob' },
    ]);
};

post '/users' => sub {
    my $c = shift;
    my $user = $c->req->json;
    $user->{id} = 3;
    $c->render(json => $user, status => 201);
};

get '/users/:id' => sub {
    my $c = shift;
    my $id = $c->param('id');

    if ($id == 1) {
        $c->render(json => { id => 1, name => 'Alice' });
    } else {
        $c->render(json => { error => 'Not found' }, status => 404);
    }
};

# Test the API
my $t = Test::Mojo->new;

# Test GET /users
$t->get_ok('/users')
  ->status_is(200)
  ->header_is('Content-Type' => 'application/json;charset=UTF-8')
  ->json_is('/0/id' => 1)
  ->json_is('/0/name' => 'Alice')
  ->json_has('/1')
  ->json_hasnt('/2');

# Test POST /users
$t->post_ok('/users' => json => { name => 'Charlie' })
  ->status_is(201)
  ->json_is('/id' => 3)
  ->json_is('/name' => 'Charlie');

# Test GET /users/:id
$t->get_ok('/users/1')
  ->status_is(200)
  ->json_is('/name' => 'Alice');

$t->get_ok('/users/999')
  ->status_is(404)
  ->json_is('/error' => 'Not found');

# Test with custom headers
$t->get_ok('/users' => { 'X-Custom-Header' => 'test' })
  ->status_is(200);

# Deep comparison
$t->get_ok('/users')
  ->json_is(bag(
      { id => 1, name => 'Alice' },
      { id => 2, name => 'Bob' },
  ));

done_testing();
```

## Rate Limiting and Throttling

Protect your APIs from abuse:

```perl
#!/usr/bin/env perl
use Mojolicious::Lite -signatures;
use Cache::FastMmap;
use Time::HiRes qw(time);

# Initialize rate limiter cache
my $cache = Cache::FastMmap->new(
    share_file => '/tmp/rate_limit.cache',
    cache_size => '1m',
    expire_time => 3600,
);

# Rate limiting middleware
helper rate_limit => sub ($c, $key, $max_requests, $window) {
    my $now = time();
    my $window_start = int($now / $window) * $window;
    my $cache_key = "$key:$window_start";

    # Get current count
    my $count = $cache->get($cache_key) || 0;

    if ($count >= $max_requests) {
        my $retry_after = $window_start + $window - $now;
        $c->res->headers->header('X-RateLimit-Limit' => $max_requests);
        $c->res->headers->header('X-RateLimit-Remaining' => 0);
        $c->res->headers->header('X-RateLimit-Reset' => $window_start + $window);
        $c->res->headers->header('Retry-After' => int($retry_after));

        $c->render(
            json => { error => 'Rate limit exceeded' },
            status => 429
        );
        return 0;
    }

    # Increment counter
    $cache->set($cache_key => ++$count);

    # Set headers
    $c->res->headers->header('X-RateLimit-Limit' => $max_requests);
    $c->res->headers->header('X-RateLimit-Remaining' => $max_requests - $count);
    $c->res->headers->header('X-RateLimit-Reset' => $window_start + $window);

    return 1;
};

# Apply rate limiting to routes
under sub ($c) {
    my $ip = $c->tx->remote_address;
    return $c->rate_limit("ip:$ip", 100, 60);  # 100 requests per minute
};

get '/api/data' => sub ($c) {
    # Additional stricter limit for this endpoint
    my $ip = $c->tx->remote_address;
    return unless $c->rate_limit("ip:$ip:data", 10, 60);  # 10 requests per minute

    $c->render(json => { data => 'sensitive information' });
};

app->start;
```

## Best Practices

1. **API Design**
   - Use consistent naming conventions
   - Version your APIs (/api/v1, /api/v2)
   - Implement proper HTTP status codes
   - Support pagination for large datasets

2. **Security**
   - Always use HTTPS in production
   - Implement authentication and authorization
   - Validate all input data
   - Rate limit to prevent abuse

3. **Performance**
   - Cache responses when appropriate
   - Use connection pooling
   - Implement request/response compression
   - Monitor API metrics

4. **Error Handling**
   - Return consistent error formats
   - Include helpful error messages
   - Log errors for debugging
   - Implement circuit breakers for external APIs

5. **Documentation**
   - Generate OpenAPI/Swagger specs
   - Provide code examples
   - Document rate limits and quotas
   - Keep changelog of API changes

## Summary

Perl's web service capabilities have evolved tremendously. With modern frameworks like Mojolicious and comprehensive HTTP libraries, Perl excels at both consuming and providing APIs. Whether you're building microservices, integrating with third-party APIs, or creating real-time applications, Perl provides the tools and flexibility you need.

The combination of CPAN's vast ecosystem, Perl's text processing prowess, and modern async capabilities makes it an excellent choice for API development in 2025. As distributed systems become ever more prevalent, Perl continues to prove its worth as a reliable, performant, and productive language for web services.