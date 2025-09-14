# Chapter 2: Getting Started - Modern Perl Setup

> "A good workman never blames his tools, but a smart one makes sure they're sharp." - Anonymous SysAdmin

Before we dive into Perl's syntax and philosophy, let's get your environment set up properly. If you're going to be a Perl programmer in 2025, you might as well do it right from the start. This chapter will save you hours of frustration and teach you how the pros manage their Perl installations.

## The Perl That's Already There (And Why You Shouldn't Use It)

Open a terminal on almost any Unix-like system and type:

```bash
perl -v
```

Congratulations! You probably have Perl installed. On macOS, Linux, BSD—it's there. This is both a blessing and a curse.

The blessing: Perl is so useful for system tasks that OS vendors include it by default.

The curse: That system Perl is there for the OS, not for you. It might be ancient (I'm looking at you, CentOS), it definitely has modules the system depends on, and updating it could break things in spectacular ways.

**Golden Rule #1**: Never mess with system Perl. Leave it alone. It's not yours.

## Enter Perlbrew: Your Personal Perl Paradise

The solution? Install your own Perl. And the easiest way to do that is with Perlbrew—a tool that lets you install and manage multiple Perl versions without sudo, without conflicting with system Perl, and without tears.

### Installing Perlbrew

On macOS with Homebrew:
```bash
brew install perlbrew
```

On Linux/Unix:
```bash
curl -L https://install.perlbrew.pl | bash
```

Now add this to your shell configuration (`~/.bashrc`, `~/.zshrc`, etc.):
```bash
source ~/perl5/perlbrew/etc/bashrc
```

Restart your terminal or run:
```bash
exec $SHELL
```

### Your First Personal Perl

Let's install a fresh, modern Perl:

```bash
# See available versions
perlbrew available

# Install the latest stable version (as of 2025, this would be 5.38+)
perlbrew install perl-5.38.2

# Or install with threading support (useful for some modules)
perlbrew install perl-5.38.2 -Dusethreads

# This will take a few minutes. Grab coffee. Perl is compiling.
```

Once installed:
```bash
# List your installed Perls
perlbrew list

# Switch to your new Perl
perlbrew switch perl-5.38.2

# Verify
perl -v
which perl  # Should show something like /home/you/perl5/perlbrew/perls/perl-5.38.2/bin/perl
```

Boom! You now have your own Perl that you can upgrade, modify, and experiment with without fear.

## CPAN: Your New Best Friend

CPAN (Comprehensive Perl Archive Network) is Perl's killer feature. It's a massive repository of reusable code that's been solving problems since 1995. But first, we need to configure it properly.

### First-Time CPAN Setup

Run:
```bash
cpan
```

The first time you run CPAN, it'll ask if you want automatic configuration. Say yes. It's smart enough to figure out your system.

But we can do better. Let's install `cpanminus` (cpanm), a more modern, zero-configuration CPAN client:

```bash
# Install cpanminus
cpan App::cpanminus

# Or, if you prefer a one-liner:
curl -L https://cpanmin.us | perl - App::cpanminus
```

Now you can install modules with:
```bash
cpanm Module::Name
```

No configuration dialogs, no interactive prompts, just installation. This is how we'll install modules throughout this book.

### Essential Modern Perl Modules

Let's install some modules that make Perl more pleasant in 2025:

```bash
# Modern::Perl - Enable modern Perl features with one line
cpanm Modern::Perl

# Perl::Tidy - Format your code consistently
cpanm Perl::Tidy

# Perl::Critic - Lint your code for best practices
cpanm Perl::Critic

# Data::Dumper - Inspect data structures (probably already installed)
cpanm Data::Dumper

# Try::Tiny - Better exception handling
cpanm Try::Tiny

# Path::Tiny - File path manipulation done right
cpanm Path::Tiny

# JSON::XS - Fast JSON parsing
cpanm JSON::XS
```

## Your First Modern Perl Script

Let's write a simple script using modern Perl features. Create a file called `hello_modern.pl`:

```perl
#!/usr/bin/env perl
use Modern::Perl '2023';
use feature 'signatures';
no warnings 'experimental::signatures';

sub greet($name = 'World') {
    say "Hello, $name!";
}

greet();
greet('Perl Hacker');

# Let's use some modern features
my @languages = qw(Perl Python Ruby Go Rust);
say "Languages I respect: " . join(', ', @languages);

# Postfix dereferencing (Perl 5.20+)
my $hashref = { name => 'Larry', language => 'Perl' };
say "Creator: " . $hashref->%*{'name'};

# State variables (like static in C)
sub counter {
    state $count = 0;
    return ++$count;
}

say "Counter: " . counter() for 1..3;
```

Run it:
```bash
perl hello_modern.pl
```

Notice what we did:
- `use Modern::Perl '2023'` enables strict, warnings, and modern features
- Subroutine signatures (no more `my ($arg1, $arg2) = @_`)
- `say` instead of `print` (automatic newline)
- State variables that maintain their value between calls
- Postfix dereferencing for cleaner syntax

## Setting Up Your Editor

You can write Perl in any text editor, but some setup makes life easier. Here are configurations for popular editors:

### VS Code
Install the "Perl" extension by Gerald Richter. It provides:
- Syntax highlighting
- Code formatting via Perl::Tidy
- Linting via Perl::Critic
- Debugger support

### Vim/Neovim
Add to your `.vimrc`:
```vim
" Perl-specific settings
autocmd FileType perl setlocal tabstop=4 shiftwidth=4 expandtab
autocmd FileType perl setlocal cindent
autocmd FileType perl setlocal cinkeys-=0#

" Run perltidy on save (optional)
autocmd BufWritePre *.pl :%!perltidy -q
```

### Emacs
Emacs has excellent Perl support out of the box with `cperl-mode`:
```elisp
(defalias 'perl-mode 'cperl-mode)
(setq cperl-indent-level 4)
(setq cperl-close-paren-offset -4)
```

## Creating a Perl Project Structure

Unlike some languages, Perl doesn't enforce a project structure. But here's a sensible layout for modern Perl projects:

```
my-perl-project/
├── lib/           # Your modules
├── script/        # Executable scripts
├── t/             # Tests
├── cpanfile       # CPAN dependencies
├── .perltidyrc    # Code formatting config
├── .perlcriticrc  # Linting config
└── README.md
```

### Managing Dependencies with cpanfile

Create a `cpanfile` to track your project's dependencies:

```perl
requires 'Modern::Perl', '1.20230701';
requires 'Try::Tiny', '0.31';
requires 'Path::Tiny', '0.144';

on 'test' => sub {
    requires 'Test::More', '1.302195';
    requires 'Test::Exception', '0.43';
};
```

Install dependencies:
```bash
cpanm --installdeps .
```

### Code Formatting with Perl::Tidy

Create `.perltidyrc`:
```
# Indent style
--indent-columns=4
--continuation-indentation=4

# Whitespace
--add-whitespace
--noblanks-before-blocks
--blanks-before-subs

# Line length
--maximum-line-length=100

# Braces
--opening-brace-on-new-line
--closing-brace-else-on-same-line
```

Format your code:
```bash
perltidy script.pl
```

### Linting with Perl::Critic

Create `.perlcriticrc`:
```
severity = 3
theme = core

[TestingAndDebugging::RequireUseStrict]
severity = 5

[TestingAndDebugging::RequireUseWarnings]
severity = 5
```

Check your code:
```bash
perlcritic script.pl
```

## The Perl Development Workflow

Here's a typical workflow for developing Perl scripts in 2025:

1. **Start with a shebang and Modern::Perl**
   ```perl
   #!/usr/bin/env perl
   use Modern::Perl '2023';
   ```

2. **Write your code**

3. **Format with perltidy**
   ```bash
   perltidy -b script.pl  # -b backs up original
   ```

4. **Check with perlcritic**
   ```bash
   perlcritic script.pl
   ```

5. **Test thoroughly** (we'll cover testing in a later chapter)

6. **Document as you go** (POD - Plain Old Documentation)

## Common Gotchas and Solutions

### Problem: "Can't locate Module/Name.pm in @INC"
**Solution**: You forgot to install the module. Run `cpanm Module::Name`

### Problem: Script works on your machine but not on the server
**Solution**: Check Perl versions (`perl -v`) and installed modules. Use `cpanfile` to track dependencies.

### Problem: "Perl is too old" errors
**Solution**: That's why we installed our own Perl with Perlbrew!

### Problem: Different behavior on different systems
**Solution**: Always use `#!/usr/bin/env perl`, not `#!/usr/bin/perl`. This uses the Perl in your PATH.

## Your Toolkit Is Ready

You now have:
- A modern Perl installation you control
- CPAN modules at your fingertips
- An editor configured for Perl development
- Tools for formatting and linting your code
- A sensible project structure

This is the foundation every serious Perl programmer needs. You're not writing CGI scripts in 1999; you're writing modern, maintainable Perl in 2025.

In the next chapter, we'll dive into Perl's data types and variables. But unlike other tutorials, we'll focus on what makes Perl's approach unique and powerful for system administration and text processing tasks.

---

*Pro tip: Bookmark [metacpan.org](https://metacpan.org). It's the modern web interface to CPAN with better search, documentation, and dependency information. When you need a module for something, start there.*