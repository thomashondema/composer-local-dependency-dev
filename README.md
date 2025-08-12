# Managing Multiple Composer Packages in a Single Repository

## Introduction
While building complex projects, we like to break up our code into single-purpose reusable components.
Not only does this make the code easier to reuse, it also makes it easier to maintain and test by promoting the 
separation of concern within our application.

In the PHP ecosystem, this is usually done by creating packages that can be installed via Composer.
For example, a web API project can use a database library, a web framework, and several other third-party 
vendor packages. Adding on top of it some code to define how to handle the API endpoints and the business logic 
behind it, which defines how the API behaves.

As the project expands to include not just a web API but also a web UI, mobile app, and CLI tool, the codebase is divided into API-specific code and the core business logic. This business logic is shared and reused by the different components of the platform.

This is a good practice, but it can lead to a codebase that is divided into many separate dependencies that will 
require development in tandem. Let's say you add a new endpoint to the API that allows to create a new user. The web API code needs to be updated to define a new route and a controller method to handle the request. This controller method should only be concerned with validating the request and responding with the appropriate HTTP status code and response body. The actual logic of creating a new user should be handled by the business logic package, which will be responsible for validating the input, checking if the user already exists, and creating the user in the database.

In practice, this means that the developer makes the necessary changes to the business logic package, commits the 
changes, releases a new version, and then updates the web API package to use the new version of the business logic 
package. If the new feature in the business logic package is not automatically tested before being committed and a bug is discovered, the developer will have to fix the bug in the business logic package, commit the changes, and then update the web API package again. This can lead to a lot of back and forth between the two packages, which can be time-consuming and error-prone. Even if the developer is careful and covers the new features with tests extensively, there is still a chance the developer will only remember to add all the correct fields to the newly introduced User model after working on the web API's validation.

To be able to skip the creation of a new release with every change we can temporarily change the required version of 
our dependency to `dev-feature-branch` in the `composer.json` file of the web API package and run a `composer 
update` to pull in the latest committed changes. This allows us to work on the business logic package and the web 
API package at the same time and when we are done, we can commit the changes to the business logic package and 
create a new release. Now we switch back to the web API package and update the required version of the business 
logic before we release this as well. While this works, it is not the most efficient way to work.

This article will explore three different methods to manage multiple Composer packages in a single repository.

## Composer Packages
Let's first remind ourselves of Composer's basic inner workings. Composer is a tool that allows you to manage 
packages and their dependencies. The two most important types of packages are project packages and library packages.
Project packages are the ones that you create for your own projects, while library packages are the ones that you 
would typically use as a dependency in your project or for another library package.
Each package has a `composer.json` file that defines the package's metadata, dependencies, and other information. 
Part of the metadata is the package's name, which is usually in the format of `vendor/package-name` and this is used to identify the package in the Composer ecosystem.

### Composer.lock
When you run `composer install` or `composer update`, Composer will create a `composer.lock` file that contains a 
snapshot of the dependencies that were installed. This file is used to ensure that the same versions of the 
dependencies are installed when you run `composer install` in the future. Generally, you should not edit this file 
manually and only let Composer manage it. For project packages this file is committed to version control, while 
for library packages it is usually not committed to not constrain the package to a specific version of its 
dependencies more than necessary.

### Repositories
By default, Composer uses packagist.org to find and install packages. Additionally, you can define your own 
repositories in your `composer.json` file to point to custom locations where your packages are hosted or even paths 
on the local file system.

## Method # 1: Nested Composer Packages in a Single Git Repository
Our company Hyperlink Industries starts off small with a single web API project. As we begin building the project, 
our goal is to keep the codebase organized and maintainable without creating a separate repository for each service, 
which would incur unnecessary overhead. Instead, we will create a single Git repository that contains all the code 
for our web API project, but we will create a separate nested package where all business logic will go. We then tell 
composer to treat this nested package as a separate Composer repository by defining it in the `composer.json` file of 
our web API project.

```
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

While installing hyperlink-industries/business-logic Composer symlinks `packages/business-logic` from 
`packages/business-logic` which results in the following folder structure:
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
Changes made in the `packages/business-logic` directory will be reflected in the 
`vendor/hyperlink-industries/business-logic` and by consequence picked up by the Composer autoloader.

## Method # 2: Using Git Submodules or Git Subtree

Our company Hyperlink Industries has grown, and we now need multiple projects that share the same business logic 
package. 

Git lets you nest one repository inside another as a subdirectory. It is a way Git projects can manage project 
dependencies without relying on tools like Composer, and it is independent of the used programming languages. However, 
it is limited to only including a specific commit of the submodule repository and in PHP will cause conflicts when 
the project depends on multiple packages that depend on different versions of the same submodule.
Doing this on a project that will not be used as a library and with limited dependencies managed as submodules should not cause any conflicts. 

### An example using Git submodules
A .gitmodules file is used to define the submodule and its location in the repository. The submodule can then be initialized and updated using Git commands.
```
[submodule "business-logic"]
    path = packages/business-logic
    url =  https://git.example.com/hyperlink-industries/business-logic.git
```

Our folder structure will not change much but the `vendor` directory will now contain a symlink to the submodule repository instead of the package installed by Composer.

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

## Method # 3: Using Composer plugins

The final method is to directly change how Composer works by using a Composer plugin. In this demonstration, we will use 
the [sandersander/composer-link](https://github.com/SanderSander/composer-link) which can be globally installed or on a per-project basis.

Composer-link is a Composer plugin that inserts itself after the `install` and `update` commands to create symlinks 
for packages that are registered with the composer-link command. This allows you to replace the installed packages 
with local versions without modifying the `composer.json` file or the `composer.lock` file. However, before deployment 
care must be taken to ensure that the code is tested with the correct versions of the dependencies defined in the `composer.lock` file.

To start of we change our `composer.json` file to use the git repository as the source for the business logic package.
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
After running `composer install`, the business logic package will be installed in the `vendor` directory as usual 
but now directly from the version control repository. This is also reflected in the `composer.lock` file so while deploying for 
production, it will not look for a local package but instead will use the version control repository.

```
"packages": [
     {
         "name": "hyperlink-industries/business-logic",
         "version": "v1.0.0",
         "source": {
             "type": "git",
             "url": "https://git.example.com/hyperlink-industries/business-logic.git",
             "reference": "74b42c25051dc2527483a58e94a995913fd53eda"
         },
         "type": "library",
         "autoload": {
             "psr-4": {
                 "HyperlinkIndustries\\BusinessLogic\\": "src/"
             }
         },
         "transport-options": {
             "relative": true
         }
     }
 ]
```


If a developer wants to work on the business logic package, they can clone the repository and link it to the project using the composer-link command.
```bash
git clone https://git.example.com/hyperlink-industries/api.git ./packages/business-logic

composer global require hyperlink-industries/composer-link
composer global link ./packages/business-logic
```
Composer-link will create a symlink in the `vendor` directory that points to the local package directory while 
leaving the `composer.lock` file intact.
Because the developer can selectively link packages, this method is more flexible than the previous two methods.



## Conclusion
In our effort to manage multiple packages in a single repository, we have explored three different methods:

Method 1 is a lightweight low effort way of separating the codebase into multiple packages to promote separation of 
concern early on in the a projects lifecycle. It is easy to set up and does not require any additional tools or configuration.
However, it does not allow for easy reusing code across multiple projects and can lead to a monolithic codebase if not appropriately managed.

Method 2 is still lightweight but adds the ability to reuse code across multiple projects by putting them in a separate repository. 
However, it does not allow for easy versioning of the packages and can lead to conflicts if multiple packages depend on different versions of the same package.

Method 3 is a more robust way of managing multiple packages in a single repository. It allows for easy versioning of the packages and can be used to manage dependencies across multiple projects. However, it requires additional tools and configuration, and can lead to conflicts if multiple packages depend on different versions of the same package.

In conclusion, the choice of method depends on the specific needs of the project and the team's workflow. Personally,
I prefer Method 3 as it allows for easy versioning of the packages and can be used to manage dependencies across 
multiple projects. However, for smaller projects or projects that do not require complex dependencies, I use Method 1.


## Notable alternatives
For those who like to not leave a single stone unturned, there are a few more options to explore:

Maintain a [separate compose.json](https://stackoverflow.com/a/59757746) for development and production environments.

Symlink the packages in the `vendor` directory to the local package directory. This can be automated with [a script](https://gist.github.com/thomashondema/5ae7c51945006e9c76cae55ca36fbc7c) that runs after the `composer install` command.

Define path repositories in the [global composer config](https://prinsfrank.nl/2019/12/27/Using-composer-to-manage-local-dev-paths) and script reverting the `composer.lock` file.

Install with [preferred-install as source](https://getcomposer.org/doc/06-config.md#preferred-install) to make sure you get a clone in your vendor directory. Now you can make changes and develop directly in the vendor directory. Composer update/install might overwrite your changes, so be careful, and push commits frequently.