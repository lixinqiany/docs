# Background

It's recommended to keep separate Git usernames and email addresses for your personal repositories and for those owned by your organizations. Otherwise, your work identity may appear in the commit history of your own GitHub projects, which is at least discomfortable when you review development statistics later. For example, the same developer appears to be committing with two different accounts, but he may forget that and be unsure in future. Importantly, it’s something I explicitly forbid on the projects I manage.

# Resolution

1. Run `git config --list` to see which identity Git will use when committing in this repository.
2. If `user.name` or `user.email` don’t match what you expect, set them explicitly with `git config user.name "..."` and `git config user.email "..."`.
