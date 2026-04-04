---
title: "groovyenv"
description: "Manage multiple Groovy versions on your machine"
layout: landing
github: "https://github.com/latte-java/groovyenv"
---

## Install

```bash
curl -fsSL https://lattejava.org/groovyenv/install | bash
```

## Usage

```bash
# Install the latest Groovy 4
groovyenv install 4

# Install a specific version
groovyenv install 4.0.26

# List available versions
groovyenv install --list

# Set the global Groovy version
groovyenv global 4

# Set the local Groovy version for the current directory
groovyenv local 4

# Set Groovy version for the current shell session (requires groovyenv init)
groovyenv shell 4

# Clear the shell version override
groovyenv shell --unset

# Show the currently active version
groovyenv current

# List installed versions
groovyenv versions

# Print GROOVY_HOME for the current version
groovyenv home

# Remove an installed version
groovyenv uninstall 4.0.26
```

## Version Resolution

groovyenv resolves the Groovy version using this precedence:

1. `GROOVYENV_VERSION` env var (set by `groovyenv shell`)
2. `.groovyversion` file in the current directory (searching up the tree)
3. `~/.groovyversion` (set by `groovyenv global`)

The file can contain a full version (`4.0.26`) or just a major version (`4`). A major version resolves to the highest installed patch for that major.

## GROOVY_HOME

To set `GROOVY_HOME` automatically and enable `groovyenv shell`, add one of the following to your shell init file:

**bash** (`~/.bashrc`):
```bash
eval "$(groovyenv init)"
```

**zsh** (`~/.zshrc`):
```zsh
eval "$(groovyenv init)"
```

**fish** (`~/.config/fish/config.fish` or Oh My Fish `init.fish`):
```fish
groovyenv init | source
```
