# Efficient Workflows for Multiple Composer Packages
<!--- 
Meta Title:
Efficient Workflows for Multiple Composer Packages: 3 Proven Methods + Alternatives

Meta Description:
Learn how to manage multiple Composer packages efficiently using path repositories, Git submodules, and Composer plugins. Includes practical alternatives, examples, and tips for projects of any size.
--->


## The Multi-Package Development Problem
As projects grow, it’s common to split code into single-purpose, reusable components. In PHP, these are often packaged for installation via Composer. For example, a web API might use a database library, a web framework, and custom packages for core business logic.

Over time, the same business logic may be reused by a web UI, mobile app, or CLI tool. This separation improves maintainability and testability, but it also creates friction. When a change in one package (e.g., adding a new API endpoint) requires updates in another, developers can get stuck in a slow cycle of commits, releases, and version bumps.

One workaround is to point `composer.json` to a `dev-feature-branch` or local path to work on multiple packages simultaneously. While this speeds up development, it can be clunky and error-prone at scale.

This article explores three efficient ways to manage multiple Composer packages, whether in a single repository or across several, so you can choose the approach that best fits your team and project stage.

## Composer Packages
Composer is PHP’s dependency manager for installing, updating, and organizing code packages.

There are two main types:

- Project packages – applications you build and run directly.

- Library packages – reusable components installed as dependencies.

Each package has a `composer.json` defining its name (`vendor/package-name`), dependencies, and autoloading rules.

### Composer.lock
`composer.lock` stores the exact dependency versions installed.

- Commit it for projects to ensure consistency across environments.

- Skip it for libraries to keep them version-flexible.

### Repositories
Composer uses Packagist.org by default, but you can define custom repositories in composer.json including private servers, VCS repositories, or local paths.



## Method #1: Nested Composer Packages in a Single Git Repository

**Scenario**  
Our company, Hyperlink Industries, starts with a single web API project. To keep the code organized without the overhead of multiple repositories, we decide to keep everything in one Git repo but separate the business logic into its own Composer package.

**How it works**  
We place the business logic in a `packages/business-logic` folder and tell Composer to treat it as a separate package by adding a `path` repository entry in the main `composer.json`.

**Example**
```json
{
    "name": "hyperlink-industries/api",
    "type": "project",
    "autoload": {
        "psr-4": {
            "HyperlinkIndustries\\Api\\": "src/"
        }
    },
    "minimum-stability": "dev",
    "require": {
        "hyperlink-industries/business-logic": "v1.0"
    },
    "repositories": [
        {
            "type": "path",
            "url": "./packages/business-logic"
        }
    ]
}
```

**Folder structure**
<!--- tree --dirsfirst --charset=ascii -a --->
```
.
|-- .git
|-- packages
|   `-- business-logic
|       |-- src
|       `-- composer.json
|-- src
|-- vendor
|   `-- hyperlink-industries
|       `-- business-logic -> ../../packages/business-logic/
|-- composer.json
`-- composer.lock
```
Changes in `packages/business-logic` are instantly reflected in `vendor/hyperlink-industries/business-logic` and picked up by the autoloader.

**When to use**  
Ideal for small to medium projects with one team. Minimal setup, no extra tooling, and quick iteration. However, it’s harder to share code across unrelated projects.


---

## Method #2: Using Git Submodules or Git Subtree

**Scenario**  
Hyperlink Industries grows and now has multiple projects that share the same business logic package. We want to keep the business logic in its own repository but still develop it alongside other projects.

**How it works**  
We use Git submodules (or subtree) to embed the business logic repository into each project’s `packages` folder. This works outside of Composer and ensures the code is synced to a specific commit.

**Example**  
A `.gitmodules` file defines the submodule:
```
[submodule "business-logic"]
    path = packages/business-logic
    url = https://git.example.com/hyperlink-industries/business-logic.git
```

Composer is still configured to load it as a local path:
```json
{
    "require": {
        "hyperlink-industries/business-logic": "v1.0"
    },
    "repositories": [
        {
            "type": "path",
            "url": "./packages/business-logic"
        }
    ]
}
```

**Folder structure**
```
.
|-- .git
|-- packages
|   `-- business-logic
|       |-- .git
|       |-- src
|       `-- composer.json
|-- src
|-- vendor
|   `-- hyperlink-industries
|       `-- business-logic -> ../../packages/business-logic/
|-- .gitmodules
|-- composer.json
`-- composer.lock
```

**When to use**  
Good for multi-project setups where you want each project to point to a specific commit of a shared repository. Requires developers to be comfortable managing submodules and doesn’t handle multiple versions of the same package well.


---

## Method #3: Using Composer Plugins

**Scenario**  
As the team expands, not everyone works on the same packages. Managing submodules becomes cumbersome, and we need a way to selectively work on local packages without changing `composer.json` or `composer.lock`.

**How it works**  
We use a Composer plugin like [sandersander/composer-link](https://github.com/SanderSander/composer-link) to override installed packages with local versions via symlinks, without altering dependency definitions. This allows per-developer flexibility.

**Example**
In `composer.json`, point the package to its Git repository:
```json
{
    "require": {
        "hyperlink-industries/business-logic": "v1.0"
    },
    "repositories": [
        {
            "type": "vcs",
            "url": "https://git.example.com/hyperlink-industries/business-logic.git"
        }
    ]
}
```

Install the plugin and link the local package:
```bash
composer global require sandersander/composer-link
git clone https://git.example.com/hyperlink-industries/business-logic.git ./packages/business-logic
composer global link ./packages/business-logic
```

**When to use**  
Best for large, long-term projects with multiple teams where some developers need to work locally on certain packages without touching others. Allows versioning and parallel package development, but requires an extra tool and care to ensure production matches tested versions.

## Conclusion
Managing multiple Composer packages in a single repository can be approached in several ways, each with its own trade-offs.

- **Method 1 (Path repositories)** – Best for small to medium projects with a single team. Minimal setup, fast iteration, but limited cross-project reuse.
- **Method 2 (Git submodules/subtree)** – Useful when code must be shared across multiple projects without heavy Composer dependency juggling. Requires comfort with Git submodules and has versioning limitations.
- **Method 3 (Composer plugins like composer-link)** – Ideal for larger teams and long-term projects where packages are developed in parallel, versioning matters, and cross-project reuse is frequent.

A practical approach is to **start simple**:
1. Begin with Method 1 during early development to keep momentum high.
2. Transition to Method 3 as your codebase grows, your packages mature, and multiple projects or teams need to work on them simultaneously.

Choosing the right method isn’t about picking one forever, it’s about matching your workflow to your current scale and evolving the approach as your project evolves.

## Try It in Your Own Projects
Personally, I start with **Method 1 (Path repositories)** for speed and simplicity in the early stages.  
Once code sharing between projects becomes important, I switch to **Method 2 (Git submodules or subtree)** to keep shared packages in their own repositories while still working on them locally.

If you want to see these approaches in action, check out my example repository here:  
[**thomashondema/composer-local-dependency-dev**](https://github.com/thomashondema/composer-local-dependency-dev)

Give the methods a try, adapt them to your workflow, and let me know which approach works best for your team.


## Alternative Composer Workflows
If none of the main methods fit perfectly, there are other ways to streamline local development with multiple Composer packages:

### 1. Separate configuration files
**Maintain a separate `composer.json` for dev and production** – e.g., `composer.dev.json` with path repositories for local development, merged via `composer install --no-scripts -d ./`.
    - **When to use:** If you want dev-only flexibility without risking production accidentally pointing to local code.
    - [Example from Stack Overflow](https://stackoverflow.com/a/59757746)

### 2. Symlinking directly
**Symlink vendor packages to local directories** – either manually or with a post-install script.
    - Can be automated with [this script](https://gist.github.com/thomashondema/5ae7c51945006e9c76cae55ca36fbc7c).
    - **When to use:** Quick-and-dirty local overrides without changing `composer.json`. Works well for solo projects or prototypes.

### 3. Global Composer config tweaks
**Define path repositories in the global Composer config** so they’re available to all projects on your machine.
    - [Guide here](https://prinsfrank.nl/2019/12/27/Using-composer-to-manage-local-dev-paths)
    - **When to use:** If you frequently work on the same local packages across multiple projects.

### 4. Preferred install as source
**Use `preferred-install: source`** in `composer.json` to get a git clone in `vendor/` instead of a dist package.
    - Lets you edit code directly in `vendor/` and commit changes upstream.
    - **When to use:** Experimental fixes or small tweaks to dependencies when you don’t want a full dev linking setup.

### 5. Use `path` repositories with `symlink: false`
Even with path repositories, you can set `"options": { "symlink": false }` to copy files instead of linking.
    - **When to use:** If symlinked packages cause IDE or tooling issues but you still want local development.

### 6. Multi-package monorepo tools
Use a dedicated tool like [monorepo-builder](https://github.com/symplify/monorepo-builder) or [composer-monorepo-plugin](https://github.com/beberlei/composer-monorepo-plugin) to manage package splitting, tagging, and dependency resolution in one repository.
    - **When to use:** Large codebases with many interdependent packages that still need to be versioned and released independently.
