---
title: "Latte Java"
description: "Latte aims to make Java simple and easy to use"
---

### 1. Install Java

```bash
curl -fsSL https://lattejava.org/javaenv/install | bash
```

```bash
javaenv install 25
```

```bash
javaenv global 25
```

### 2. Install Latte

```bash
curl -fsSL https://lattejava.org/cli/install | bash
```

### 3. Create a project

```bash
mkdir my-project && cd my-project
```

```bash
latte init
```

_Write some code...._

### 4. Login into Latte

```bash
latte login
```

### 5. Create a Group

Visit https://app.lattejava.org/app/groups/new to create your Group in the Latte repository.

_If you use a reverse-DNS Group, you'll also need to verify your domain._

### 6. Release & public the project

```bash
latte release
```

That's it!

Your project will now have a tag based on the version in `project.latte` and it will be released to the Latte repository for the world to enjoy.