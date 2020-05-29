## Introduction
The aim of this guide is to provide just enough detail to get you productive when working with [Go Modules](https://golang.org/cmd/go/#hdr-Modules__module_versions__and_more). It is derived from a [series of articles on Go Modules, written by Russ Cox](https://research.swtch.com/vgo), the creator of Go Modules.

If you would like to delve deeper into the design considerations behind Go Modules, please refer to the [Go Modules Proposal](https://go.googlesource.com/proposal/+/master/design/24301-versioned-go.md).

If you would like to learn about all the commands, flags and options related to Go Modules, then the [Go Documentation](https://golang.org/cmd/go/#hdr-Modules__module_versions__and_more) and the [Go blog articles on Go Modules](https://blog.golang.org/using-go-modules) may be the right place to start.

???+ tip "Spare me the lecture. I need to quickly learn how to:"
    - [make the current directory the root of a module](#how-can-i-make-the-current-directory-the-root-of-a-module)
    - [add a dependency](https://golang.org/cmd/go/#hdr-Add_dependencies_to_current_module_and_install_them)
    - [set a dependency to a specific version(upgrade/downgrade)](#how-does-the-mvs-algorithm-upgrade-one-module-to-a-specific-newer-version)
    - [update all dependencies](#how-does-the-mvs-algorithm-upgrade-all-modules-to-their-latest-version)
    - [clean up unused dependencies](#how-do-i-clean-up-unused-dependencies)
    - [set up the correct environment variables](#how-do-i-know-which-environment-variable-to-use-when-working-with-go-modules)
    - [fix my failing builds](#help-my-build-is-failing-and-i-dont-know-where-to-start)
    
    
## What are Go Modules?
In Go, programs are constructed by linking together packages. A package is built from one or more source files that together declare the `constants`, `types`, `variables` and `functions` that belong to it. These declarations are accessible by all files of the same package, and also exported for use in other packages.

Go Modules introduce the concept of a module, defined as a collection of Go packages that are stored in a file tree, with a [`go.mod`](https://golang.org/pkg/cmd/go/#hdr-The_go_mod_file ) file at its root. This `go.mod` file defines the path for accessing the module (serving as its [import path](https://golang.org/cmd/go/#hdr-Import_path_syntax)) and lists the module's dependency requirements, which are the other modules needed to successfully build the module. 

Each dependency is written as a module path and a specific semantic version, in a [semantic version string](https://golang.org/cmd/go/#hdr-Module_compatibility_and_semantic_versioning) format.

Concretely, a Go Module is a group of packages versioned as a single unit, with a `go.mod` file that declares the minimum requirements that must be satisfied by the module's dependencies, and where the group of packages share a common import path prefix.

In addition to `go.mod`, Go Modules introduce a [`go.sum`](https://golang.org/pkg/cmd/go/#hdr-Module_authentication_using_go_sum) file, which contains the cryptographic hashes of the content of specific module versions. When Go downloads each module, it computes a hash of the file tree corresponding to that module. The hash and the version of the module are included in the binary. The Go command then uses the `go.sum` file to ensure that future downloads of a module retrieve the same bits as the first download, thus guaranteeing that the modules a project depends on, do not change unexpectedly. Additionally, Go Modules introduce an [auditable checksum database](https://go.googlesource.com/proposal/+/master/design/25530-sumdb.md) which will be used by the `go` command to authenticate modules.

## Why Go Modules?
Go Modules bring package versioning to Go. Versioning enables reproducible builds by ensuring that a program builds exactly the same way tomorrow as it does today. Modules also adopt semantic versioning, eliminates vendoring, and deprecates `$GOPATH` in favor of a project-based workflow. 

Previously, when `go get` needed a package, it always fetched the latest copy, delegating the download and update operations to version control systems like Git. This meant that without a concept of versioning, `go get` could not convey to users any expectations about what kind of changes to expect in a given update. With versioning, library writers and users can frame the expectation around updates. Consequently there is a clear expectation on breaking updates versus non-breaking updates. 

## Why Semantic Import Versioning?
Go Modules champion the _Import Compatibility Rule_, where the expectation is that newer versions of a package, with a given import path, should be backwards-compatible with older versions. 

If an old package and a new package have the same import path, then the new package must be backwards compatible with the old package.

- ex: "github.com/heetch/foo_package@v1.2.3" => "github.com/heetch/foo_package@1.2.5"

If a breaking change is required, then a new package should be created with a new import path.

- ex: "github.com/heetch/foo_package@v1.2.3" => "github.com/heetch/foo_package/v2@v2.0.0"

Because each major version has a different import path, a module can contain one of each major version of a package. This means modules need to specify only the minimum requirements for their dependencies. Further, since both v1 and v2 of a package can coexist, this makes it easy to upgrade the clients of a package one at a time while guaranteeing successful builds. v0 is special because of its special role in [_Semantic Versioning Precedence Rules_](https://semver.org/#spec-item-11) where major versions >=1 have different import paths.

## How are the module versions to use in each build determined?
Go Modules use the [_Minimal Version Selection_](https://research.swtch.com/vgo-mvs) (MVS) algorithm to determine which module versions to use in each build. A build of a module by itself uses the specific versions of required dependencies listed in the go.mod file. 

However, in larger builds, it only uses a newer version if a dependency in the build requires it.
[_Minimal Version Selection_](https://research.swtch.com/vgo-mvs) always selects the minimal (oldest) module version that satisfies the overall requirements of a build. Thus, the release of a new version has no effect on the build.

1. It assumes that each module declares its own dependency requirements (as a list of the minimum versions of other modules)
2. It assumes that modules follow the [_Import Compatibility Rule_](https://research.swtch.com/vgo-import) where packages in newer versions should work as well as the older versions.

As such, the dependency requirement guarantees only a minimum version, never a maximum version or a list of incompatible later versions.

## How does the Minimal Version Selection (MVS) algorithm construct the build list for a given module?
To construct the build list for a given module, the MVS algorithm starts the list with the module itself, and then appends each dependency's own build list. If a module appears in the list multiple times, it keeps only the newest module version. 

The build list is constructed using a graph reachability algorithm, with the given module as the starting node. The algorithm recursively traverses the graph, taking care not to visit a node that has already been visited. Once the traversal is complete (once it has a rough build list of nodes, including nodes with multiple versions), it computes the final build list by keeping only the newest version of any listed module. All other versions in the rough build list are discarded.

## How does the MVS algorithm upgrade all modules to their latest version?
To upgrade all modules to their latest versions, the MVS algorithm computes an upgraded build list by updating the module requirement graph so that every outbound link to any version of a module is replaced with one that links to the latest version of that module. It then uses the [previous algorithm](#how-does-the-minimal-version-selection-mvs-algorithm-construct-the-build-list-for-a-given-module) to determine the final build list.

**Commands:**

- `go get -u`(without any arguments) => upgrade direct and indirect dependencies of the current package
- `go get -u ./...` (from the module root) => upgrade all direct and indirect dependencies of your module, excluding test dependencies
- `go get -u=patch ./...`(from the module root) => same as above, using the latest patch release
- `go get -u -t ./...`(from the module root) => upgrade all direct and indirect dependencies of your module, including test dependencies
- `go get -u=patch -t ./...`(from the module root) => same as above, using the latest patch release
- `go get -u all` => update all of the packages transitively imported by the main module (including test dependencies)
- `go get -u github.com/heetch/foo` => get the latest version of foo as well as the latest versions for all of the direct and indirect dependencies of `foo`

## How does the MVS algorithm upgrade one module to a specific newer version?
To upgrade one module to a specific newer version, it first constructs the build list of the non-upgraded module as before. Then it appends the new module's build list. If a module appears in the list multiple times, it keeps only the newest module version.

**Commands:**

- `go get foo@v1.2.3` => get a specific version
- `go get foo@latest` => get the latest version 
- `go get foo@master` => using a branch name such as `@master` obtains the latest commit regardless of whether or not it has a [semantic versioning tag](#why-semantic-import-versioning)

???+ tip "Common Workflow"
    A common starting point when upgrading `foo` is:

        - `go get foo` or `go get foo@latest` and after things are working
        - `go get -u=patch foo` to upgrade all direct and indirect dependencies of `foo`

## How does the MVS algorithm downgrade one module to a specific older version?
To downgrade one module to a specific older version, it rewinds the required version of each top-level dependency, until that dependency's build list no longer refers to newer versions of the downgraded module. Downgrading one module may require downgrading other modules, so the algorithm tries to downgrade as few other modules as possible.

If a requirement is incompatible with the proposed downgrade (if the requirement's build list includes a now-disallowed module version), then it successively tries older versions until it finds one that is compatible with the downgrade.

**Commands:**

- `go get foo@v1.2.3` => downgrade the `foo` package to the specified version

## What advantage does this have over systems that use _lock_ files?
Because a module's dependency requirements are included with the module's source code to uniquely determine how to build it, and because the [_Minimal Version Selection_ algorithm](#how-does-the-minimal-version-selection-mvs-algorithm-construct-the-build-list-for-a-given-module) provides high-fidelity builds by using the oldest version available that meets the dependency requirements, mechanisms such as _lock_ files are not needed.

In other systems, the _lock_ file lists the specific versions a build should use. It only guarantees reproducible builds for whole programs, not for library modules.

## Why does using Go Modules make vendoring unnecessary?
Previously, if you were using an external package and worried that it might change in unexpected ways, you copied it to your local repository, and stored the copy under a new import path that identified it as a local copy, for example:

- github.com/pkg => my_local_path/external/github.com/pkg

Over time, this was encapsulated in tools that copied a dependency into your repository and also updated all the import paths wihin it to reflect the new location. However, these modifications made it harder to compare against, and even incorporate newer copies, required updates, etc., to other code using that dependency. 

Package managers like `go dep` evolved to become a tool that could be used for freezing external package dependencies. This introduced the concept of _vendoring_, where you could copy dependencies into the project without modifying the source files as long as the `GOPATH` was correctly set up. But these vendoring tools relied on metadata files such as `glide.yml` or `dep.yml` to provide reproducible builds, and could not help with deciding which versions of a package to use. 

_Vendor_ directories:

- specify by their contents, the exact version of the dependencies to use when running `go build`.
- ensure the availability of those dependencies, even if the original copies disappear.
- are unwieldly and bloat the repositories in which they are used.

`GOPATH` was used to:

- define the versions of dependencies.
- hold the source code for those dependencies.
- provide a way to infer the import path for code in a specific directory.

With Go Modules, `GOPATH`s are no longer required because the `go.mod` file includes the full module path and defines the version of each dependency in use. Further, since a directory with a `go.mod` file marks the root of the directory tree, this directory serves as a self-contained workspace, and is separate from all other directories. 

## Why are pseudo-versions used to identify untagged commits?
To name untagged commits, Go Modules use the pseudo-version format `v0.0.0-yyyymmddhhmmss-{commit identifier}` to identify a specific commit made on a given date. The _commit identifier_ is typically a shorted Git hash, and must have a commit time matching the UTC timestamp. 

The pseudo-version form is used so that [_Semantic Versioning Precedence Rules_](https://semver.org/#spec-item-11) can be used to compare two pseudo-versions by commit time, since the timestamp encoding makes string comparison match time comparison. This also ensures that `go mod` always prefers a tagged semantic version(even if it's very old, such as `v0.0.1`), over an untagged pseudo-version, since it has a greater semantic version precedence than any `v0.0.0` pre-release

## Can I use Go Modules along with the traditional `GOPATH`-based mechanisms?
Currently, the Go command defaults to module mode when run in directories outside `GOPATH/src` that are marked by `go.mod` files in their roots. This can be overriden by setting the **_transitional_** environment variable `$GO111MODULE` to `on` or `off`. The default mode is `auto`.

## How can I make the current directory, the root of a module?
`go mod init` : when working in a directory outside `$GOPATH`. This command writes a `go.mod` file at the root of your directory

## Can packages also have `go.mod` files?
The `go.mod` file only appears in the root of the module. Packages in subdirectories have import paths consisting of the module path and the path to that subdirectory.

## What happens if a package is not provided by any module in `go.mod`?
The Go command automatically looks up the module containing that package and adds it to `go.mod` using the _latest_ version. _Latest_ is defined as the latest tagged stable version, or else, the latest tagged pre-release version, or else, the latest untagged version. Only direct dependencies are recorded in the `go.mod` file.

## Can modules be cached locally?
Yes, they are cached locally in `$GOPATH/pkg/mod`

## How are different major versions of package distinguished?
Every major version of a `Go Module` _`{v1, v2, v3, ...}`_ uses a different module path. Starting at `v2`, the module path must end in the major version. [_Semantic Import Versioning_](#why-semantic-import-versioning) is used to give incompatible packages(packages with different major versions) different names. 

## Can I build a program using different minor versions?
No, since these are considered duplicates of a single module path. Go Modules allow a build to include at most one version of any particular module path, which means, at most, one of each major version. Allowing different major versions of a module (since they have different paths) enables incremental upgrades to a new major version. 

## How do I clean up unused dependencies?
`go mod tidy`

## What does `go mod tidy` do?
`go mod tidy` finds all the packages transitively imported by packages in your module and adds new module requirements for packages not provided by any known modules, while removing requirements on modules that do not provide any imported packages. If a module provides packages that are only imported by projects that have not yet migrated to modules, it marks that module requirement with an `// indirect` comment.

## I get errors when I run `go mod tidy`, what could it be?
- When `go mod tidy` adds a requirement, it adds the latest version of the module. However, if your `$GOPATH` included an older version of a dependency that subsequently published a breaking change,`go mod tidy` will be unable to resolve the right package. 
- Run `go mod graph` to get a graph of the module dependency requirements and then try downgrading to an older version with the `go get` command.

## How can I fix tests that are failing because they cannot write files in the package directory?
Tests may fail when the package directory is in the module cache, which has read-only access. Configure your tests to copy the files they need to write to a temporary directory instead. 

## How can I fix tests that are failing because they rely on relative paths to packages in another module
Tests that use relative paths to locate and read files in another package can fail if the package resides in a different module. You can either:

- copy the test inputs into your module
- convert the test inputs from raw files to data embedded in `.go` source files

## How can I fix tests that are failing becase they expect `go` commands within the test to run in `$GOPATH` mode?
- Add a `go.mod` file to the source tree to be tested
- Set `GO111MODULE=off`

## How do I release a new module after committing my changes?
- tag your release `git tag v1.2.3`
- publish your release with the same tag `git push origin v1.2.3`

The new `go.mod` file at the root of your module defines the canonical import path for the module and adds the new minimum version requirements

## How do I know which environment variable to use when working with Go Modules?
- `GO111MODULE={on|off}` => transitional variable to toggle the use of Go Modules
- `GOPRIVATE` => controls which modules the Go command considers to be private (not available publicly) and should therefore not use the proxy or checksum database. ex: `export GOPRIVATE=github.com/heetch`
- `GONOPROXY` => overrides `GOPRIVATE` for the specific decision of whether to use a proxy. ex: `export GONOPROXY=github.com/heetch`
- `GONOSUMDB` => overrides `GOPRIVATE` for the specific decision of whether to use the checksum database. ex: `export GONOSUMDB=github.com/heetch`

## Help! My build is failing and I don't know where to start!
- Ensure that the following environment variables are appropriately set: `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB`
- Run [`go mod tidy`](#what-does-go-mod-tidy-do)
- Run `go mod graph` to [make sense of the module dependency requirements](#how-are-the-module-versions-to-use-in-each-build-determined)
- Apply your understanding of how dependency requirements are resolved based on the following scenarios:
    - [upgrading all modules](#how-does-the-mvs-algorithm-upgrade-all-modules-to-their-latest-version)
    - [upgrading one module](#how-does-the-mvs-algorithm-upgrade-one-module-to-a-specific-newer-version)
    - [downgrading one module](#how-does-the-mvs-algorithm-downgrade-one-module-to-a-specific-older-version)
    - If all else fails, someone in the `#dev-go-talk` may be able to help

## Reference
- [original series of articles on Go Modules, written by Russ Cox](https://research.swtch.com/vgo), the creator of Go Modules
- [Go blog articles on Go Modules](https://blog.golang.org/using-go-modules)


???+ tip "Need help?"

    - Post your question on these Slack channels:
      -  Dev General @ `#dev-general`


    **Did you find this page useful?**

    :heart_eyes:󠀠󠀠 [**Yep!**](https://forms.gle/DPKLG2KpNnam79VNA)

    :weary: [**No I did not**](https://forms.gle/DPKLG2KpNnam79VNA)


**Maintained by:** Kelvin Spencer
