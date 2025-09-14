# Chapter 8: Working with CSV, JSON, and XML

> "Data formats are like opinions - everyone has one, and they all stink except yours." - DevOps Proverb

In a perfect world, all data would be in one format. In reality, you're juggling CSV exports from Excel, JSON from REST APIs, and XML from that enterprise system installed in 2003. Perl doesn't judge—it parses them all. This chapter shows you how to read, write, and transform between these formats without losing your sanity.

## CSV: The Deceptively Simple Format

CSV looks simple. It's just commas and values, right? Wrong. There are quotes, escaped quotes, embedded newlines, different delimiters, encodings, and Excel's creative interpretations. Don't write your own CSV parser. Use Text::CSV.

### Reading CSV Files

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use Text::CSV;

# Basic CSV reading
my $csv = Text::CSV->new({ binary => 1, auto_diag => 1 });

open my $fh, '<:encoding(utf8)', 'data.csv' or die $!;

# Read header row
my $headers = $csv->getline($fh);
$csv->column_names(@$headers);

# Read data rows as hashrefs
while (my $row = $csv->getline_hr($fh)) {
    say "Processing user: $row->{username}";
    say "  Email: $row->{email}";
    say "  Status: $row->{status}";
}

close $fh;

# Or slurp all at once
sub read_csv {
    my ($filename) = @_;

    my $csv = Text::CSV->new({
        binary => 1,
        auto_diag => 1,
        sep_char => ',',  # Could be ';' or '\t' for TSV
        quote_char => '"',
        escape_char => '"',
        allow_loose_quotes => 1,
        allow_loose_escapes => 1,
    });

    open my $fh, '<:encoding(utf8)', $filename or die $!;

    # Read all rows
    my $rows = $csv->getline_all($fh);
    close $fh;

    # Convert to array of hashrefs
    my $headers = shift @$rows;
    my @data = map {
        my %hash;
        @hash{@$headers} = @$_;
        \%hash;
    } @$rows;

    return \@data;
}
```

### Writing CSV Files

```perl
# Write CSV with proper escaping
sub write_csv {
    my ($filename, $data, $headers) = @_;

    my $csv = Text::CSV->new({
        binary => 1,
        auto_diag => 1,
        eol => "\n",  # Auto line endings
    });

    open my $fh, '>:encoding(utf8)', $filename or die $!;

    # Write headers if provided
    if ($headers) {
        $csv->say($fh, $headers);
    } elsif (@$data && ref $data->[0] eq 'HASH') {
        # Extract headers from first hashref
        my @headers = sort keys %{$data->[0]};
        $csv->say($fh, \@headers);

        # Write data rows
        for my $row (@$data) {
            $csv->say($fh, [@$row{@headers}]);
        }
    } else {
        # Array of arrays
        $csv->say($fh, $_) for @$data;
    }

    close $fh;
}

# Example: Generate report
my @report_data = (
    { date => '2024-01-15', sales => 1500, region => 'North' },
    { date => '2024-01-15', sales => 2300, region => 'South' },
    { date => '2024-01-16', sales => 1800, region => 'North' },
);

write_csv('sales_report.csv', \@report_data);
```

### Handling Problematic CSV

```perl
# Deal with Excel's CSV quirks
sub parse_excel_csv {
    my ($filename) = @_;

    # Excel likes to add BOM
    open my $fh, '<:encoding(utf8)', $filename or die $!;
    my $first_line = <$fh>;

    # Remove BOM if present
    $first_line =~ s/^\x{FEFF}//;

    seek($fh, 0, 0);  # Rewind

    my $csv = Text::CSV->new({
        binary => 1,
        auto_diag => 1,
        # Excel sometimes uses semicolons in some locales
        sep_char => index($first_line, ';') > -1 ? ';' : ',',
        allow_whitespace => 1,  # Handle extra spaces
        blank_is_undef => 1,    # Empty cells become undef
    });

    # Process file...
}

# Handle malformed CSV
sub parse_dirty_csv {
    my ($filename) = @_;
    my @clean_data;

    open my $fh, '<', $filename or die $!;

    while (my $line = <$fh>) {
        chomp $line;

        # Try to parse with Text::CSV first
        my $csv = Text::CSV->new({ binary => 1 });

        if ($csv->parse($line)) {
            push @clean_data, [$csv->fields];
        } else {
            # Fallback to manual parsing for broken lines
            warn "Failed to parse line $.: " . $csv->error_input;

            # Simple split (dangerous but sometimes necessary)
            my @fields = split /,/, $line;
            s/^\s+|\s+$//g for @fields;  # Trim
            push @clean_data, \@fields;
        }
    }

    close $fh;
    return \@clean_data;
}
```

## JSON: The Modern Standard

JSON is everywhere. REST APIs speak it, configuration files use it, and JavaScript loves it. Perl handles JSON beautifully with JSON::XS (fast) or JSON::PP (pure Perl).

### Reading and Writing JSON

```perl
use JSON::XS;

# Create JSON encoder/decoder
my $json = JSON::XS->new->utf8->pretty->canonical;

# Decode JSON
my $json_text = '{"name":"Alice","age":30,"active":true}';
my $data = $json->decode($json_text);
say "Name: $data->{name}";

# Encode to JSON
my $perl_data = {
    servers => [
        { name => 'web01', ip => '10.0.1.1', status => 'running' },
        { name => 'web02', ip => '10.0.1.2', status => 'running' },
        { name => 'db01',  ip => '10.0.2.1', status => 'stopped' },
    ],
    updated => time(),
    version => '1.0',
};

my $json_output = $json->encode($perl_data);
print $json_output;

# File operations
sub read_json {
    my ($filename) = @_;

    open my $fh, '<:encoding(utf8)', $filename or die $!;
    local $/;  # Slurp mode
    my $json_text = <$fh>;
    close $fh;

    return decode_json($json_text);  # Using functional interface
}

sub write_json {
    my ($filename, $data) = @_;

    my $json = JSON::XS->new->utf8->pretty->canonical;

    open my $fh, '>:encoding(utf8)', $filename or die $!;
    print $fh $json->encode($data);
    close $fh;
}
```

### Advanced JSON Handling

```perl
# Custom JSON encoder settings
my $json = JSON::XS->new
    ->utf8(1)           # Encode/decode UTF-8
    ->pretty(1)         # Human-readable output
    ->canonical(1)      # Sort keys for consistent output
    ->allow_nonref(1)   # Allow non-reference values
    ->allow_blessed(1)  # Allow blessed references
    ->convert_blessed(1) # Call TO_JSON method on objects
    ->max_depth(512)    # Prevent deep recursion
    ->max_size(10_000_000); # 10MB max

# Handle special values
my $data = {
    string => "Hello",
    number => 42,
    float => 3.14,
    true => \1,  # JSON::XS::true
    false => \0, # JSON::XS::false
    null => undef,
    unicode => "Hello 世界 🌍",
};

# Streaming JSON parser for large files
sub process_json_stream {
    my ($filename, $callback) = @_;

    open my $fh, '<:encoding(utf8)', $filename or die $!;

    my $json = JSON::XS->new->utf8;
    my $parser = JSON::XS->new->utf8->incremental;

    while (my $chunk = <$fh>) {
        $parser->incr_parse($chunk);

        # Extract complete JSON objects
        while (my $obj = $parser->incr_parse) {
            $callback->($obj);
        }
    }

    close $fh;
}

# JSON with schema validation
use JSON::Validator;

my $validator = JSON::Validator->new;
$validator->schema({
    type => 'object',
    required => ['name', 'email'],
    properties => {
        name => { type => 'string', minLength => 1 },
        email => { type => 'string', format => 'email' },
        age => { type => 'integer', minimum => 0, maximum => 150 },
    },
});

my @errors = $validator->validate({
    name => 'Alice',
    email => 'alice@example.com',
    age => 30,
});

die "Validation failed: @errors" if @errors;
```

### JSON Transformations

```perl
# Convert between JSON and other formats
sub csv_to_json {
    my ($csv_file, $json_file) = @_;

    # Read CSV
    my $csv = Text::CSV->new({ binary => 1, auto_diag => 1 });
    open my $fh, '<:encoding(utf8)', $csv_file or die $!;

    my $headers = $csv->getline($fh);
    $csv->column_names(@$headers);

    my @data;
    while (my $row = $csv->getline_hr($fh)) {
        push @data, $row;
    }
    close $fh;

    # Write JSON
    write_json($json_file, \@data);
}

# Transform JSON structure
sub transform_json {
    my ($data) = @_;

    # Flatten nested structure
    my @flattened;
    for my $user (@{$data->{users}}) {
        for my $order (@{$user->{orders}}) {
            push @flattened, {
                user_id => $user->{id},
                user_name => $user->{name},
                order_id => $order->{id},
                order_total => $order->{total},
                order_date => $order->{date},
            };
        }
    }

    return \@flattened;
}
```

## XML: The Enterprise Format

XML is verbose, complex, and everywhere in enterprise systems. Perl has excellent XML support through various modules. We'll focus on XML::LibXML (fast and standards-compliant) and XML::Simple (for simple cases).

### XML::Simple for Basic Tasks

```perl
use XML::Simple;
use Data::Dumper;

# Read simple XML
my $xml_text = <<'XML';
<config>
    <server name="web01" ip="10.0.1.1" status="active"/>
    <server name="web02" ip="10.0.1.2" status="active"/>
    <server name="db01" ip="10.0.2.1" status="maintenance"/>
</config>
XML

my $data = XMLin($xml_text,
    ForceArray => ['server'],  # Always make array
    KeyAttr => { server => 'name' },  # Use 'name' as hash key
);

print Dumper($data);

# Write XML
my $output = XMLout($data,
    RootName => 'config',
    NoAttr => 0,  # Use attributes
    XMLDecl => '<?xml version="1.0" encoding="UTF-8"?>',
);

print $output;
```

### XML::LibXML for Serious XML Work

```perl
use XML::LibXML;

# Parse XML
my $parser = XML::LibXML->new();
my $doc = $parser->parse_string($xml_text);

# Or parse from file
my $doc = $parser->parse_file('config.xml');

# XPath queries
my @servers = $doc->findnodes('//server[@status="active"]');
for my $server (@servers) {
    say "Active server: " . $server->getAttribute('name');
    say "  IP: " . $server->getAttribute('ip');
}

# Modify XML
my ($db_server) = $doc->findnodes('//server[@name="db01"]');
$db_server->setAttribute('status', 'active');
$db_server->appendTextChild('note', 'Maintenance completed');

# Create new XML
my $new_doc = XML::LibXML::Document->new('1.0', 'UTF-8');
my $root = $new_doc->createElement('inventory');
$new_doc->setDocumentElement($root);

for my $item (@inventory) {
    my $elem = $new_doc->createElement('item');
    $elem->setAttribute('id', $item->{id});
    $elem->setAttribute('quantity', $item->{quantity});
    $elem->appendTextChild('name', $item->{name});
    $elem->appendTextChild('price', $item->{price});
    $root->appendChild($elem);
}

print $new_doc->toString(1);  # Pretty print
```

### Parsing Complex XML

```perl
# Parse XML with namespaces
my $xml_with_ns = <<'XML';
<root xmlns:app="http://example.com/app">
    <app:user id="1">
        <app:name>Alice</app:name>
        <app:email>alice@example.com</app:email>
    </app:user>
</root>
XML

my $doc = $parser->parse_string($xml_with_ns);
my $xpc = XML::LibXML::XPathContext->new($doc);
$xpc->registerNs('app', 'http://example.com/app');

my @users = $xpc->findnodes('//app:user');
for my $user (@users) {
    my $name = $xpc->findvalue('app:name', $user);
    my $email = $xpc->findvalue('app:email', $user);
    say "User: $name <$email>";
}

# Stream parsing for large XML
sub parse_large_xml {
    my ($filename, $callback) = @_;

    my $reader = XML::LibXML::Reader->new(location => $filename)
        or die "Cannot read $filename";

    while ($reader->read) {
        if ($reader->nodeType == XML_READER_TYPE_ELEMENT
            && $reader->name eq 'record') {

            my $node = $reader->copyCurrentNode(1);
            $callback->($node);
        }
    }
}

# Validate against XSD
sub validate_xml {
    my ($xml_file, $xsd_file) = @_;

    my $schema_doc = $parser->parse_file($xsd_file);
    my $schema = XML::LibXML::Schema->new(string => $schema_doc);

    my $doc = $parser->parse_file($xml_file);

    eval { $schema->validate($doc) };
    if ($@) {
        die "Validation failed: $@";
    }

    say "XML is valid!";
}
```

## Format Conversion Utilities

### Universal Data Converter

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

package DataConverter;
use Text::CSV;
use JSON::XS;
use XML::Simple;
use YAML::XS;

sub new($class) {
    return bless {}, $class;
}

sub convert($self, $input_file, $output_file, $from_format, $to_format) {
    my $data = $self->read($input_file, $from_format);
    $self->write($output_file, $data, $to_format);
}

sub read($self, $file, $format) {
    my $method = "read_$format";
    die "Unknown format: $format" unless $self->can($method);
    return $self->$method($file);
}

sub write($self, $file, $data, $format) {
    my $method = "write_$format";
    die "Unknown format: $format" unless $self->can($method);
    return $self->$method($file, $data);
}

sub read_json($self, $file) {
    open my $fh, '<:encoding(utf8)', $file or die $!;
    local $/;
    my $json = <$fh>;
    close $fh;
    return decode_json($json);
}

sub write_json($self, $file, $data) {
    my $json = JSON::XS->new->utf8->pretty->canonical;
    open my $fh, '>:encoding(utf8)', $file or die $!;
    print $fh $json->encode($data);
    close $fh;
}

sub read_csv($self, $file) {
    my $csv = Text::CSV->new({ binary => 1, auto_diag => 1 });
    open my $fh, '<:encoding(utf8)', $file or die $!;

    my $headers = $csv->getline($fh);
    $csv->column_names(@$headers);

    my @data;
    while (my $row = $csv->getline_hr($fh)) {
        push @data, $row;
    }
    close $fh;

    return \@data;
}

sub write_csv($self, $file, $data) {
    my $csv = Text::CSV->new({ binary => 1, auto_diag => 1, eol => "\n" });
    open my $fh, '>:encoding(utf8)', $file or die $!;

    # Assume array of hashrefs
    if (@$data && ref $data->[0] eq 'HASH') {
        my @headers = sort keys %{$data->[0]};
        $csv->say($fh, \@headers);

        for my $row (@$data) {
            $csv->say($fh, [@$row{@headers}]);
        }
    }
    close $fh;
}

sub read_xml($self, $file) {
    return XMLin($file,
        ForceArray => 1,
        KeyAttr => [],
    );
}

sub write_xml($self, $file, $data) {
    my $xml = XMLout($data,
        RootName => 'data',
        XMLDecl => '<?xml version="1.0" encoding="UTF-8"?>',
    );

    open my $fh, '>:encoding(utf8)', $file or die $!;
    print $fh $xml;
    close $fh;
}

sub read_yaml($self, $file) {
    return YAML::XS::LoadFile($file);
}

sub write_yaml($self, $file, $data) {
    YAML::XS::DumpFile($file, $data);
}

package main;

# Use the converter
my $converter = DataConverter->new();

# Convert CSV to JSON
$converter->convert('data.csv', 'data.json', 'csv', 'json');

# Convert JSON to XML
$converter->convert('data.json', 'data.xml', 'json', 'xml');

# Convert XML to YAML
$converter->convert('data.xml', 'data.yaml', 'xml', 'yaml');
```

## Real-World Example: API Data Pipeline

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use LWP::UserAgent;
use JSON::XS;
use Text::CSV;
use Try::Tiny;

# Fetch data from API, process, and export
sub api_data_pipeline {
    my ($api_url, $output_file) = @_;

    # Fetch from API
    my $ua = LWP::UserAgent->new(timeout => 30);
    my $response = $ua->get($api_url);

    die "API request failed: " . $response->status_line
        unless $response->is_success;

    # Parse JSON response
    my $data = decode_json($response->content);

    # Transform data
    my $processed = process_api_data($data);

    # Export to multiple formats
    export_data($processed, $output_file);
}

sub process_api_data {
    my ($data) = @_;

    my @processed;
    for my $item (@{$data->{results}}) {
        push @processed, {
            id => $item->{id},
            name => clean_text($item->{name}),
            value => sprintf("%.2f", $item->{value} || 0),
            timestamp => format_timestamp($item->{created_at}),
            status => normalize_status($item->{status}),
        };
    }

    return \@processed;
}

sub clean_text {
    my ($text) = @_;
    $text =~ s/^\s+|\s+$//g;  # Trim
    $text =~ s/\s+/ /g;       # Normalize spaces
    return $text;
}

sub format_timestamp {
    my ($ts) = @_;
    # Convert various timestamp formats to ISO 8601
    # Implementation depends on input format
    return $ts;
}

sub normalize_status {
    my ($status) = @_;
    my %status_map = (
        active => 'active',
        enabled => 'active',
        disabled => 'inactive',
        deleted => 'archived',
    );
    return $status_map{lc($status)} // 'unknown';
}

sub export_data {
    my ($data, $base_filename) = @_;

    # Export as JSON
    my $json_file = "$base_filename.json";
    write_json($json_file, $data);
    say "Exported to $json_file";

    # Export as CSV
    my $csv_file = "$base_filename.csv";
    write_csv($csv_file, $data);
    say "Exported to $csv_file";

    # Generate summary
    generate_summary($data, "$base_filename.summary.txt");
}

sub generate_summary {
    my ($data, $filename) = @_;

    open my $fh, '>:encoding(utf8)', $filename or die $!;

    my %stats = (
        total => scalar(@$data),
        by_status => {},
    );

    for my $item (@$data) {
        $stats{by_status}{$item->{status}}++;
    }

    print $fh "Data Export Summary\n";
    print $fh "=" x 40 . "\n";
    print $fh "Total Records: $stats{total}\n\n";
    print $fh "By Status:\n";
    for my $status (sort keys %{$stats{by_status}}) {
        printf $fh "  %-15s: %d\n", $status, $stats{by_status}{$status};
    }

    close $fh;
    say "Summary written to $filename";
}
```

## Performance Considerations

1. **Choose the right parser** - XML::LibXML is faster than XML::Simple
2. **Use streaming for large files** - Don't load 1GB XML into memory
3. **Prefer JSON::XS over JSON::PP** - 10-100x faster
4. **Cache parsed data** - Parse once, use many times
5. **Validate incrementally** - Don't wait until the end to find errors
6. **Use binary mode for CSV** - Handles all encodings properly
7. **Consider format limitations** - CSV can't represent nested data well

## Best Practices

1. **Always validate input** - Never trust external data
2. **Handle encoding explicitly** - UTF-8 everywhere
3. **Use appropriate error handling** - Malformed data is common
4. **Document format expectations** - Include sample files
5. **Test with real-world data** - Edge cases are everywhere
6. **Provide format detection** - Auto-detect when possible
7. **Keep original data** - Transform copies, not originals
8. **Version your schemas** - Data formats evolve

## Conclusion

Data comes in many formats, but Perl handles them all with aplomb. Whether you're parsing gigabytes of CSV, streaming JSON from APIs, or wrestling with enterprise XML, Perl has the tools you need. The key is choosing the right tool for each format and understanding its quirks.

Remember: every data format has edge cases. Plan for them, test for them, and handle them gracefully. Your future self will thank you when that "perfectly formatted" data file turns out to be anything but.

---

*Next: Log file analysis and monitoring. We'll build real-time log processors that can handle anything from Apache access logs to custom application outputs.*