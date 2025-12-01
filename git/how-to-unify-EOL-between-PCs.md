## Background

In a development team, it is common for people to work on different operating systems (e.g. macOS and Windows). These systems use different newline characters by default: LF on Unix-like systems (including macOS) and CRLF on Windows. If a repository does not define a consistent policy for end of lines via a `.gitattributes` file, each developer's local `core.autocrlf` setting can cause Git to convert line endings differently.

A typical failure scenario looks like this:

- One developer on Windows commits files that currently use CRLF line endings.
- Another developer on macOS or Linux pulls the code, edits the same files, and their editor saves them with LF line endings.
- When that developer creates a merge request, `git diff` shows almost every line as changed, which makes code review painful and `git blame` much less useful.

To avoid this, the team needs a single, repository-level rule that defines how line endings should be stored and checked out, regardless of the contributor's operating system.

## Solution

The recommended way to enforce consistent line endings across all contributors is to use a `.gitattributes` file in the repository. Attribute rules in `.gitattributes` take precedence over each developer's local `core.autocrlf` configuration.

The `core.autocrlf` setting is a per-machine Git configuration that controls how Git converts line endings when moving files between the repository and the working tree:

- `true` (commonly used on Windows): on checkout, Git converts LF → CRLF; on commit, Git converts CRLF → LF, so the repository always stores LF.
- `input` (commonly used on Unix-like systems): Git keeps LF in the working tree and converts CRLF → LF on commit; it does not convert LF → CRLF on checkout.
- `false`: Git does not perform any automatic line-ending conversion.

The syntax in `.gitattributes` is:

`<pattern> <attr1> <attr2> <attr3>...`

Here, `pattern` matches file paths using Git's path pattern syntax (similar to shell globs, not regular expressions), and each `attr` sets one or more attributes for matching files.

To normalize line endings for all text files in a repository and force LF everywhere, you can add the following rule to `.gitattributes`:

```bash
# eol = End Of Line
* text=auto eol=lf
```

This configuration means:

- `*`: apply the rule to all files in the repository.
- `text=auto`: let Git automatically detect whether a file is text; for text files, normalize line endings to LF in the repository.
- `eol=lf`: when checking files out into the working tree, always use LF line endings, regardless of the contributor's operating system or their `core.autocrlf` value.

Tip:
- You can use `git config --global core.safecrlf true` to forbid commits with mixture of new line characters.
