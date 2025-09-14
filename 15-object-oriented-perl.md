# Chapter 15: Object-Oriented Perl

> "Perl's OO is like duct tape - it might not be pretty, but it holds everything together and gets the job done." - Anonymous

Perl's approach to object-oriented programming is unique: it gives you the tools to build any OO system you want, rather than forcing one paradigm. From basic blessed references to sophisticated metaprogramming with Moose, this chapter covers the full spectrum of OOP in Perl, with a focus on practical, maintainable code.

## Classic Perl OOP

### Blessed References

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

# Basic class definition
package Point {
    # Constructor
    sub new {
        my ($class, $x, $y) = @_;
        my $self = {
            x => $x // 0,
            y => $y // 0,
        };
        return bless $self, $class;
    }

    # Accessor methods
    sub x {
        my ($self, $value) = @_;
        $self->{x} = $value if defined $value;
        return $self->{x};
    }

    sub y {
        my ($self, $value) = @_;
        $self->{y} = $value if defined $value;
        return $self->{y};
    }

    # Methods
    sub move {
        my ($self, $dx, $dy) = @_;
        $self->{x} += $dx;
        $self->{y} += $dy;
        return $self;
    }

    sub distance_to {
        my ($self, $other) = @_;
        my $dx = $self->{x} - $other->{x};
        my $dy = $self->{y} - $other->{y};
        return sqrt($dx * $dx + $dy * $dy);
    }

    # String representation
    sub to_string {
        my $self = shift;
        return "Point($self->{x}, $self->{y})";
    }

    # Destructor
    sub DESTROY {
        my $self = shift;
        # Cleanup code here
    }
}

# Usage
my $p1 = Point->new(3, 4);
my $p2 = Point->new(0, 0);

say $p1->to_string();                    # Point(3, 4)
say "Distance: " . $p1->distance_to($p2); # Distance: 5

$p1->move(1, 1);
say $p1->to_string();                    # Point(4, 5)
```

### Inheritance

```perl
package Shape {
    sub new {
        my ($class, %args) = @_;
        return bless \%args, $class;
    }

    sub area {
        die "Subclass must implement area()";
    }

    sub perimeter {
        die "Subclass must implement perimeter()";
    }

    sub describe {
        my $self = shift;
        return ref($self) . " with area " . $self->area();
    }
}

package Rectangle {
    use parent 'Shape';  # Inheritance

    sub new {
        my ($class, $width, $height) = @_;
        my $self = $class->SUPER::new(
            width => $width,
            height => $height,
        );
        return $self;
    }

    sub area {
        my $self = shift;
        return $self->{width} * $self->{height};
    }

    sub perimeter {
        my $self = shift;
        return 2 * ($self->{width} + $self->{height});
    }
}

package Circle {
    use parent 'Shape';
    use constant PI => 3.14159;

    sub new {
        my ($class, $radius) = @_;
        my $self = $class->SUPER::new(radius => $radius);
        return $self;
    }

    sub area {
        my $self = shift;
        return PI * $self->{radius} ** 2;
    }

    sub perimeter {
        my $self = shift;
        return 2 * PI * $self->{radius};
    }
}

# Polymorphism
my @shapes = (
    Rectangle->new(5, 3),
    Circle->new(4),
    Rectangle->new(2, 8),
);

for my $shape (@shapes) {
    say $shape->describe();
}
```

### Encapsulation with Closures

```perl
package Counter {
    sub new {
        my ($class, $initial) = @_;
        $initial //= 0;

        # Private variable in closure
        my $count = $initial;

        # Return blessed coderef with methods
        return bless {
            increment => sub { ++$count },
            decrement => sub { --$count },
            get => sub { $count },
            reset => sub { $count = $initial },
        }, $class;
    }

    # Method dispatch
    sub AUTOLOAD {
        my $self = shift;
        my $method = our $AUTOLOAD;
        $method =~ s/.*:://;

        return if $method eq 'DESTROY';

        if (exists $self->{$method}) {
            return $self->{$method}->(@_);
        }

        die "Unknown method: $method";
    }
}

# Usage
my $counter = Counter->new(10);
say $counter->get();        # 10
$counter->increment();
$counter->increment();
say $counter->get();        # 12
$counter->reset();
say $counter->get();        # 10
```

## Modern OOP with Moo

### Moo Basics

```perl
package Person {
    use Moo;
    use Types::Standard qw(Str Int);

    # Attributes with type constraints
    has name => (
        is => 'ro',           # read-only
        isa => Str,
        required => 1,
    );

    has age => (
        is => 'rw',           # read-write
        isa => Int,
        default => 0,
        trigger => sub {
            my ($self, $new_age) = @_;
            warn "Age cannot be negative!" if $new_age < 0;
        },
    );

    has email => (
        is => 'rw',
        isa => Str,
        predicate => 'has_email',  # Creates has_email() method
        clearer => 'clear_email',  # Creates clear_email() method
    );

    # Lazy attribute with builder
    has id => (
        is => 'ro',
        lazy => 1,
        builder => '_build_id',
    );

    sub _build_id {
        my $self = shift;
        return sprintf("%s_%d_%d", $self->name, $self->age, time());
    }

    # Method modifiers
    before 'age' => sub {
        my ($self, $new_age) = @_;
        return unless defined $new_age;
        say "Changing age from " . $self->age . " to $new_age";
    };

    # Regular methods
    sub introduce {
        my $self = shift;
        my $intro = "Hi, I'm " . $self->name;
        $intro .= ", I'm " . $self->age . " years old" if $self->age;
        $intro .= ", email me at " . $self->email if $self->has_email;
        return $intro;
    }
}

# Usage
my $person = Person->new(
    name => 'Alice',
    age => 30,
    email => 'alice@example.com',
);

say $person->introduce();
$person->age(31);  # Triggers before modifier
say "Has email: " . ($person->has_email ? 'Yes' : 'No');
```

### Roles (Mixins)

```perl
# Define roles
package Role::Timestamped {
    use Moo::Role;
    use Time::Piece;

    has created_at => (
        is => 'ro',
        default => sub { localtime->datetime },
    );

    has updated_at => (
        is => 'rw',
        default => sub { localtime->datetime },
    );

    before [qw(update save)] => sub {
        my $self = shift;
        $self->updated_at(localtime->datetime);
    };
}

package Role::Serializable {
    use Moo::Role;
    use JSON::XS;

    requires 'to_hash';  # Consumer must implement

    sub to_json {
        my $self = shift;
        return encode_json($self->to_hash);
    }

    sub from_json {
        my ($class, $json) = @_;
        my $data = decode_json($json);
        return $class->new(%$data);
    }
}

# Use roles
package Document {
    use Moo;

    with 'Role::Timestamped', 'Role::Serializable';

    has title => (is => 'rw', required => 1);
    has content => (is => 'rw', default => '');
    has author => (is => 'ro', required => 1);

    sub to_hash {
        my $self = shift;
        return {
            title => $self->title,
            content => $self->content,
            author => $self->author,
            created_at => $self->created_at,
            updated_at => $self->updated_at,
        };
    }

    sub update {
        my ($self, %changes) = @_;
        $self->title($changes{title}) if exists $changes{title};
        $self->content($changes{content}) if exists $changes{content};
    }
}

# Usage
my $doc = Document->new(
    title => 'My Document',
    author => 'Bob',
);

$doc->update(content => 'Some content');
say $doc->to_json();
```

## Advanced OOP with Moose

### Moose Features

```perl
package Employee {
    use Moose;
    use Moose::Util::TypeConstraints;

    # Custom types
    subtype 'Email'
        => as 'Str'
        => where { /^[\w\.\-]+@[\w\.\-]+$/ }
        => message { "Invalid email address: $_" };

    subtype 'PositiveInt'
        => as 'Int'
        => where { $_ > 0 };

    # Attributes with advanced features
    has name => (
        is => 'rw',
        isa => 'Str',
        required => 1,
        documentation => 'Employee full name',
    );

    has email => (
        is => 'rw',
        isa => 'Email',
        required => 1,
    );

    has salary => (
        is => 'rw',
        isa => 'PositiveInt',
        traits => ['Counter'],
        handles => {
            increase_salary => 'inc',
            decrease_salary => 'dec',
        },
    );

    has department => (
        is => 'rw',
        isa => 'Department',
        weak_ref => 1,  # Prevent circular references
    );

    has skills => (
        is => 'rw',
        isa => 'ArrayRef[Str]',
        default => sub { [] },
        traits => ['Array'],
        handles => {
            add_skill => 'push',
            has_skills => 'count',
            list_skills => 'elements',
            find_skill => 'first',
        },
    );

    # Method modifiers
    around 'salary' => sub {
        my ($orig, $self, $new_salary) = @_;

        if (defined $new_salary) {
            my $old_salary = $self->$orig();
            $self->log_salary_change($old_salary, $new_salary);
        }

        return $self->$orig($new_salary);
    };

    # BUILD is called after object construction
    sub BUILD {
        my $self = shift;
        $self->register_employee();
    }

    # DEMOLISH is called before object destruction
    sub DEMOLISH {
        my $self = shift;
        $self->unregister_employee();
    }

    # Make immutable for performance
    __PACKAGE__->meta->make_immutable;
}

package Department {
    use Moose;

    has name => (is => 'ro', isa => 'Str', required => 1);
    has employees => (
        is => 'rw',
        isa => 'ArrayRef[Employee]',
        default => sub { [] },
        traits => ['Array'],
        handles => {
            add_employee => 'push',
            employee_count => 'count',
            all_employees => 'elements',
        },
    );

    __PACKAGE__->meta->make_immutable;
}
```

### Metaprogramming

```perl
package DynamicClass {
    use Moose;
    use Moose::Meta::Attribute;

    # Add attributes dynamically
    sub add_attribute {
        my ($self, $name, %options) = @_;

        $self->meta->add_attribute(
            $name => (
                is => 'rw',
                %options,
            )
        );
    }

    # Add methods dynamically
    sub add_method {
        my ($self, $name, $code) = @_;

        $self->meta->add_method($name => $code);
    }

    # Introspection
    sub describe {
        my $self = shift;
        my $meta = $self->meta;

        say "Class: " . $meta->name;
        say "Attributes:";
        for my $attr ($meta->get_all_attributes) {
            say "  - " . $attr->name . " (" . $attr->type_constraint . ")";
        }

        say "Methods:";
        for my $method ($meta->get_all_methods) {
            say "  - " . $method->name;
        }
    }
}

# Runtime class modification
my $obj = DynamicClass->new();

$obj->add_attribute('color', isa => 'Str', default => 'blue');
$obj->add_method('greet', sub {
    my $self = shift;
    return "Hello, I'm a " . $self->color . " object";
});

$obj->color('red');
say $obj->greet();  # Hello, I'm a red object
$obj->describe();
```

## Design Patterns in Perl

### Singleton Pattern

```perl
package Singleton {
    use Moo;

    my $instance;

    sub instance {
        my $class = shift;
        $instance //= $class->new(@_);
        return $instance;
    }

    sub new {
        my $class = shift;
        die "Use $class->instance() instead" if $instance;
        return bless {@_}, $class;
    }
}

# Usage
my $s1 = Singleton->instance();
my $s2 = Singleton->instance();
say $s1 == $s2 ? "Same instance" : "Different instances";  # Same instance
```

### Factory Pattern

```perl
package AnimalFactory {
    use Module::Runtime qw(use_module);

    my %animal_types = (
        dog => 'Animal::Dog',
        cat => 'Animal::Cat',
        bird => 'Animal::Bird',
    );

    sub create_animal {
        my ($class, $type, %args) = @_;

        my $animal_class = $animal_types{$type}
            or die "Unknown animal type: $type";

        use_module($animal_class);
        return $animal_class->new(%args);
    }
}

package Animal {
    use Moo;
    has name => (is => 'ro', required => 1);
    sub speak { die "Subclass must implement" }
}

package Animal::Dog {
    use Moo;
    extends 'Animal';
    sub speak { "Woof!" }
}

package Animal::Cat {
    use Moo;
    extends 'Animal';
    sub speak { "Meow!" }
}

# Usage
my $dog = AnimalFactory->create_animal('dog', name => 'Rex');
say $dog->speak();  # Woof!
```

### Observer Pattern

```perl
package Observable {
    use Moo::Role;

    has observers => (
        is => 'ro',
        default => sub { [] },
    );

    sub attach {
        my ($self, $observer) = @_;
        push @{$self->observers}, $observer;
    }

    sub detach {
        my ($self, $observer) = @_;
        @{$self->observers} = grep { $_ != $observer } @{$self->observers};
    }

    sub notify {
        my ($self, $event, @args) = @_;
        $_->update($self, $event, @args) for @{$self->observers};
    }
}

package StockPrice {
    use Moo;
    with 'Observable';

    has symbol => (is => 'ro', required => 1);
    has price => (is => 'rw', trigger => sub {
        my ($self, $new_price) = @_;
        $self->notify('price_changed', $new_price);
    });
}

package StockDisplay {
    use Moo;

    sub update {
        my ($self, $subject, $event, @args) = @_;
        if ($event eq 'price_changed') {
            say "Stock " . $subject->symbol . " changed to $args[0]";
        }
    }
}

# Usage
my $stock = StockPrice->new(symbol => 'AAPL');
my $display = StockDisplay->new();

$stock->attach($display);
$stock->price(150.25);  # Stock AAPL changed to 150.25
```

## Real-World OOP Example: Task Queue System

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';

package TaskQueue::Task {
    use Moo;
    use Types::Standard qw(Str Enum Int CodeRef HashRef);

    has id => (
        is => 'ro',
        default => sub {
            state $counter = 0;
            return ++$counter;
        },
    );

    has name => (
        is => 'ro',
        isa => Str,
        required => 1,
    );

    has status => (
        is => 'rw',
        isa => Enum[qw(pending running completed failed)],
        default => 'pending',
    );

    has priority => (
        is => 'ro',
        isa => Int,
        default => 0,
    );

    has handler => (
        is => 'ro',
        isa => CodeRef,
        required => 1,
    );

    has data => (
        is => 'ro',
        isa => HashRef,
        default => sub { {} },
    );

    has result => (
        is => 'rw',
    );

    has error => (
        is => 'rw',
        isa => Str,
    );

    sub execute {
        my $self = shift;

        $self->status('running');

        eval {
            $self->result($self->handler->($self->data));
            $self->status('completed');
        };

        if ($@) {
            $self->error($@);
            $self->status('failed');
            return 0;
        }

        return 1;
    }
}

package TaskQueue::Queue {
    use Moo;
    use Types::Standard qw(ArrayRef InstanceOf);

    has tasks => (
        is => 'ro',
        isa => ArrayRef[InstanceOf['TaskQueue::Task']],
        default => sub { [] },
    );

    has running => (
        is => 'rw',
        isa => ArrayRef[InstanceOf['TaskQueue::Task']],
        default => sub { [] },
    );

    has max_concurrent => (
        is => 'ro',
        default => 1,
    );

    sub add_task {
        my ($self, $task) = @_;
        push @{$self->tasks}, $task;
        $self->_sort_tasks();
    }

    sub _sort_tasks {
        my $self = shift;
        @{$self->tasks} = sort {
            $b->priority <=> $a->priority
        } @{$self->tasks};
    }

    sub get_next_task {
        my $self = shift;
        return shift @{$self->tasks};
    }

    sub process {
        my $self = shift;

        while (@{$self->tasks} || @{$self->running}) {
            # Start new tasks if under limit
            while (@{$self->tasks} &&
                   @{$self->running} < $self->max_concurrent) {
                my $task = $self->get_next_task();
                push @{$self->running}, $task;

                say "Starting task: " . $task->name;
                $task->execute();
            }

            # Clean up completed tasks
            @{$self->running} = grep {
                $_->status eq 'running'
            } @{$self->running};

            sleep(0.1) if @{$self->running};
        }
    }

    sub stats {
        my $self = shift;

        my %stats = (
            pending => 0,
            running => 0,
            completed => 0,
            failed => 0,
        );

        for my $task (@{$self->tasks}, @{$self->running}) {
            $stats{$task->status}++;
        }

        return \%stats;
    }
}

package TaskQueue::Worker {
    use Moo;
    use threads;
    use Thread::Queue;

    has queue => (
        is => 'ro',
        isa => InstanceOf['Thread::Queue'],
        default => sub { Thread::Queue->new() },
    );

    has workers => (
        is => 'ro',
        default => sub { [] },
    );

    has num_workers => (
        is => 'ro',
        default => 4,
    );

    sub start {
        my $self = shift;

        for (1..$self->num_workers) {
            my $thread = threads->create(sub {
                $self->worker_loop();
            });
            push @{$self->workers}, $thread;
        }
    }

    sub worker_loop {
        my $self = shift;

        while (my $task = $self->queue->dequeue()) {
            last if $task eq 'STOP';
            $task->execute();
        }
    }

    sub stop {
        my $self = shift;

        $self->queue->enqueue('STOP') for @{$self->workers};
        $_->join() for @{$self->workers};
    }

    sub add_task {
        my ($self, $task) = @_;
        $self->queue->enqueue($task);
    }
}

# Usage
package main;

my $queue = TaskQueue::Queue->new(max_concurrent => 3);

# Add tasks
$queue->add_task(TaskQueue::Task->new(
    name => 'High Priority Task',
    priority => 10,
    handler => sub {
        my $data = shift;
        sleep(2);
        return "Processed: $data->{input}";
    },
    data => { input => 'important data' },
));

$queue->add_task(TaskQueue::Task->new(
    name => 'Low Priority Task',
    priority => 1,
    handler => sub {
        sleep(1);
        return "Done";
    },
));

# Process queue
$queue->process();

# Check results
for my $task (@{$queue->tasks}) {
    if ($task->status eq 'completed') {
        say $task->name . ": " . $task->result;
    } elsif ($task->status eq 'failed') {
        say $task->name . " failed: " . $task->error;
    }
}
```

## Best Practices

1. **Use modern OOP modules** - Moo for lightweight, Moose for features
2. **Favor composition over inheritance** - Use roles/traits
3. **Make attributes read-only when possible** - Immutability helps
4. **Use type constraints** - Catch errors early
5. **Document your classes** - Especially public interfaces
6. **Write tests for your classes** - OOP code needs good tests
7. **Use builders for complex objects** - Separate construction logic
8. **Keep classes focused** - Single Responsibility Principle
9. **Use lazy attributes wisely** - For expensive computations
10. **Make classes immutable in production** - Better performance

## Conclusion

Perl's object-oriented programming is incredibly flexible, from simple blessed references to sophisticated metaprogramming. While the syntax might seem unusual compared to other languages, the power and flexibility it provides is unmatched. Modern tools like Moo and Moose make OOP in Perl both powerful and pleasant.

Remember: choose the right level of abstraction for your problem. Not every script needs Moose, but when you're building large applications, its features can make your code more maintainable and robust.

---

*Next: Testing and debugging. We'll explore Perl's excellent testing culture and tools for finding and fixing bugs.*