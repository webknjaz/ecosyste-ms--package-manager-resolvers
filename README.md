# Package Manager Dependency Resolution

A reference for dependency resolution algorithms and strategies across different package managers.

## Algorithm Categories

Package managers generally fall into a few algorithmic families.

Some registries only list one version per package in their index (APT, DNF, Pacman, Homebrew, Alpine), which simplifies resolution since there's no version selection. Others (npm, PyPI, RubyGems, crates.io, Maven Central) list all historical versions, requiring the resolver to choose among candidates.

Language package registries rarely remove old versions or packages. System package repositories are more curated, partly to avoid conflicts and unresolvable dependency trees.

| Algorithm Family | Description | Package Managers |
|------------------|-------------|------------------|
| **SAT Solving** | Translates dependencies to boolean satisfiability | Composer, DNF/Zypper/Conda (libsolv) |
| **ASP** | Answer Set Programming for optimization | Spack |
| **PubGrub** | Conflict-driven clause learning with good error messages | Dart pub, Poetry, uv, SwiftPM |
| **Molinillo** | Backtracking with forward checking | Bundler, CocoaPods, RubyGems |
| **Backtracking** | Try versions, backtrack on conflict | pip, Cargo, Cabal |
| **Minimal Version Selection** | Always use minimum satisfying version | Go modules |
| **Deduplication with nesting** | Deduplicate where possible, nest on conflict | npm, Yarn, pnpm, Bun |
| **Version Mediation** | Pick based on graph position or declaration order | Maven, Gradle, NuGet |
| **Scoring/Priority** | Assigns scores to packages, resolves by priority | APT/aptitude |
| **Bundled** | Dependencies included at build time, no runtime resolution | Snap |

---

## JavaScript

### npm

npm (v7+) uses Arborist for dependency resolution, building a logical graph of dependencies overlaid on a physical tree of folders.

**Algorithm**: Maximally naive deduplication with nested fallback

**How it works**:
- Builds a queue of dependencies starting from root
- For each dependency, scans upward to find the shallowest placement that causes no conflicts
- Preferentially deduplicates by finding versions that satisfy multiple dependents
- When no single version satisfies all dependents, nests different versions under their respective requesters
- Peer dependencies are validated as a set at each potential placement (they cannot be resolved by nesting)

**Trade-offs**:
- **Phantom dependencies**: Packages can accidentally access dependencies they did not declare because of hoisting
- **Doppelgangers**: The same package may exist in multiple places when conflicts occur
- **Peer dependency complexity**: Peer deps must be at same level, creating genuine conflicts that nesting cannot solve

**References**:
- [npm v7 Series - Arborist Deep Dive](https://blog.npmjs.org/post/618653678433435649/npm-v7-series-arborist-deep-dive.html)
- [Arborist on GitHub](https://github.com/npm/arborist)

---

### Yarn Classic (v1)

**Algorithm**: Similar to npm (deduplication with nested fallback)

**How it works**:
- Deduplicates where possible, nests when versions conflict
- `yarn.lock` ensures deterministic installs across machines

**Trade-offs**:
- Same phantom dependency problem as npm
- Deterministic lockfile was a key improvement over early npm

**References**:
- [Yarn Classic Documentation](https://classic.yarnpkg.com/en/docs/)

---

### Yarn Berry (v2+)

Yarn Berry has its own resolver implementation (complete rewrite from Yarn Classic).

**Algorithm**: Own implementation; deduplication with nested fallback

**How it works**:
- Own resolver in `@yarnpkg/core` ([source](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-core/sources/Resolver.ts))
- Deduplicates where possible, allows multiple versions when conflicts arise
- Installation via Plug'n'Play: generates `.pnp.cjs` lookup file instead of `node_modules`

**Trade-offs**:
- **Faster installs**: No `node_modules` tree to write
- **No phantom dependencies** (in strict mode)
- **Compatibility issues**: Some tools expect `node_modules` to exist

**References**:
- [Plug'n'Play](https://yarnpkg.com/features/pnp)
- [Resolver source](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-core/sources/Resolver.ts)

---

### pnpm

pnpm has its own resolver implementation with similar semantics to npm.

**Algorithm**: Own implementation; deduplication with isolated installation

**How it works**:
- Own resolver (`@pnpm/resolve-dependencies` and related packages)
- Deduplicates where possible, allows multiple versions when conflicts arise
- Each package gets isolated `node_modules/` with only declared dependencies

**Trade-offs**:
- **No phantom dependencies**: Packages only see their declared dependencies
- **Stricter than npm**: Some packages may break if they rely on hoisting

**References**:
- [Motivation](https://pnpm.io/motivation)
- [How peers are resolved](https://pnpm.io/how-peers-are-resolved)

---

### Bun

Bun has its own resolver implementation written in Zig.

**Algorithm**: Own implementation; npm-compatible semantics

**How it works**:
- Own resolver written in Zig
- Uses hoisted installation by default (like npm)
- Binary lockfile (`bun.lockb`) for fast parsing

**Trade-offs**:
- **Very fast**: Written in Zig with systems-level optimizations
- **Compatible with npm**: Uses same `node_modules` layout
- **No peer dependency conflict detection**: Does not warn about peer conflicts

**References**:
- [bun install](https://bun.sh/package-manager)

---

## Python

### pip

pip (v20.3+) uses a backtracking resolver based on the resolvelib library.

**Algorithm**: Backtracking with on-demand metadata fetching

**How it works**:
- Uses resolvelib, an abstract backtracking resolution algorithm
- Fetches dependency metadata lazily (on-demand) because pre-computing the full tree is too expensive
- When a conflict is found, backtracks to try alternative versions
- NP-hard problem; can be slow on complex dependency graphs

**Trade-offs**:
- **Correct but slow**: Backtracking can take a long time on complex graphs
- **Lazy metadata**: Cannot precompute optimal solution
- **Better than legacy resolver**: Less likely to break environments
- **No epochs in version comparisons**: PEP 440 epochs are supported but add complexity

**References**:
- [Dependency Resolution](https://pip.pypa.io/en/stable/topics/dependency-resolution/)
- [More on Dependency Resolution](https://pip.pypa.io/en/stable/topics/more-dependency-resolution/)

---

### Poetry

Poetry uses a PubGrub-based resolver (Mixology library).

**Algorithm**: PubGrub (conflict-driven clause learning)

**How it works**:
- Uses Mixology, a Python implementation of PubGrub
- Conflict-driven clause learning borrowed from SAT solvers
- Learns from conflicts to prune search space
- Provides clear error messages explaining why resolution failed

**Trade-offs**:
- **Better error messages**: Explains conflict chains
- **Faster than naive backtracking** on many cases
- **Slow metadata fetching**: PyPI does not always provide dependency metadata via API; Poetry must download packages to inspect
- **Constraints help**: Narrower version ranges speed up resolution

**Limitations**:
- PubGrub does not support version epochs, which is why PyPA chose resolvelib for pip

**References**:
- [Poetry FAQ](https://python-poetry.org/docs/faq/)
- [Mixology](https://github.com/sdispater/mixology)
- [pipgrip](https://github.com/ddelange/pipgrip)

---

### uv

uv (from Astral) uses pubgrub-rs, a Rust implementation of PubGrub.

**Algorithm**: PubGrub with performance optimizations

**How it works**:
- Uses pubgrub-rs for version solving
- Starts with virtual root package, selects highest priority undecided package
- When conflict occurs, learns incompatibility to avoid repeating
- Uses heuristics like backtracking and reordering packages that conflict frequently

**Trade-offs**:
- **Fast**: 8-10x faster than pip cold, 80-115x with cache
- **Good error messages**: Inherited from PubGrub
- **Drop-in replacement**: Compatible with pip/pip-tools/poetry workflows

**References**:
- [uv Resolver Internals](https://docs.astral.sh/uv/reference/internals/resolver/)
- [uv GitHub](https://github.com/astral-sh/uv)

---

### Conda / Mamba

Conda now uses libsolv (via libmamba) as its default solver. Mamba is a faster reimplementation of conda using the same solver.

**Algorithm**: SAT solving via libsolv

**How it works**:
- Uses libsolv, the same SAT solver used by DNF/Zypper
- Conflict-driven clause learning (CDCL)
- Filters and sorts by channel priority, then by version

**History**: Conda's original solver used pycosat (PicoSAT wrapper). libmamba became the default in conda 23.10 (2023) due to significant performance improvements.

**Trade-offs**:
- **Much faster than original**: 15s vs 91s in benchmarks
- **Same solver as DNF**: Well-tested libsolv

**References**:
- [Package resolution](https://mamba.readthedocs.io/en/latest/advanced_usage/package_resolution.html)
- [libmamba solver announcement](https://conda.org/blog/2023-11-06-conda-23-10-0-release/)

---

## Ruby

### Bundler / RubyGems

Bundler and RubyGems use Molinillo, a backtracking resolver with forward checking.

**Algorithm**: Molinillo (backtracking with forward checking and conflict-driven backjumping)

**How it works**:
- Maintains a stack of dependency and possibility states
- For each dependency, sorts possibilities (versions) and tries them
- Uses forward checking to detect conflicts early
- On conflict, unwinds to the earliest state that contributed to the conflict
- Shares resolver with CocoaPods

**Trade-offs**:
- **Efficient backjumping**: Skips irrelevant states on conflict
- **Possibility grouping**: Versions with same sub-dependencies grouped to reduce search
- **Shared**: Used by Bundler, RubyGems, and CocoaPods

**References**:
- [Molinillo](https://github.com/CocoaPods/Molinillo)
- [Molinillo Architecture](https://github.com/CocoaPods/Molinillo/blob/master/ARCHITECTURE.md)
- [Bundler 1.9 Announcement](https://bundler.io/blog/2015/03/21/hello-bundler-19.html)

---

## Rust

### Cargo

Cargo uses a backtracking resolver that tries the highest compatible version first.

**Algorithm**: Backtracking with semver-aware version selection

**How it works**:
- Tries highest version first (within semver bounds)
- Allows multiple major versions of the same package to coexist
- Uses `ActivationsKey` to track: only one semver-compatible version allowed
- Features are unified across the dependency graph
- `links` field ensures native libraries linked only once

**Version interpretation**:
- `1.2.3` means `>=1.2.3, <2.0.0`
- `0.2.3` means `>=0.2.3, <0.3.0`
- `0.0.3` means `>=0.0.3, <0.0.4`

**Trade-offs**:
- **Multiple major versions**: Different major versions can coexist
- **Feature unification**: All feature flags combined (can cause unexpected compilation)
- **Can produce duplicates**: Different minor versions might both be included if not deduplicable
- **DFS-based**: Can be slow on large graphs

**References**:
- [Dependency Resolution](https://doc.rust-lang.org/cargo/reference/resolver.html)
- [SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html)
- [Specifying Dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)

---

## Perl

### cpanm / Carton

cpanm installs the latest version of each dependency. Carton adds lockfile support on top of cpanm.

**Algorithm**: Latest version

**How it works**:
- Installs latest version satisfying constraints
- Single version of each module
- Carton's `cpanfile.snapshot` locks versions for reproducibility

**Trade-offs**:
- **Simple**: No complex version selection
- **Carton for reproducibility**: Snapshot file locks exact versions

**References**:
- [Carton](https://metacpan.org/pod/Carton)
- [cpanm](https://metacpan.org/pod/App::cpanminus)

---

## Go

### Go Modules

Go modules use Minimal Version Selection (MVS).

**Algorithm**: Minimal Version Selection

**How it works**:
- Does **not** select the newest version
- Selects the **minimum** version that satisfies all requirements
- Traverses the module graph, tracking maximum required version for each module
- At the end, uses the highest version seen for each module (which is the minimum that works)
- `go.sum` records cryptographic checksums for verification

**Trade-offs**:
- **Predictable**: Easy to understand and implement (~50 lines of code)
- **High-fidelity builds**: Uses versions authors tested with
- **May use old versions**: Does not automatically upgrade to latest
- **Security updates require explicit action**: Must run `go get -u` to upgrade

**References**:
- [Minimal Version Selection](https://research.swtch.com/vgo-mvs)
- [Go Modules Reference](https://go.dev/ref/mod)
- [Modules Part 03: Minimal Version Selection](https://www.ardanlabs.com/blog/2019/12/modules-03-minimal-version-selection.html)

---

## Java/JVM

### Maven

Maven uses "nearest definition first" with depth-based mediation.

**Algorithm**: Nearest definition wins (breadth-first, first declaration breaks ties)

**How it works**:
- Traverses dependency graph breadth-first
- For each dependency, uses the version from the nearest definition in the tree
- If two dependencies are at same depth, first declaration in POM wins
- `<dependencyManagement>` section takes precedence over mediation

**Trade-offs**:
- **Not newest wins**: May use older version if declared closer to root
- **Order-dependent**: Declaration order in POM affects resolution
- **Can be surprising**: Transitive dependency version may override direct
- **No configurable strategy**: Cannot switch to "highest wins" (though Maven 4 may change this)

**Workarounds**:
- Use `maven-enforcer-plugin` with `requireUpperBoundDeps` rule
- Use `<dependencyManagement>` to pin versions

**References**:
- [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
- [Dependency Mediation and Conflict Resolution](https://cwiki.apache.org/confluence/display/MAVENOLD/Dependency+Mediation+and+Conflict+Resolution)
- [How does version resolution work in Maven and Gradle?](http://jlbp.dev/how-does-version-resolution-work-in-maven-and-gradle)

---

### Gradle

Gradle defaults to "newest version wins" with configurable conflict resolution.

**Algorithm**: Highest version wins (configurable)

**How it works**:
- Collects all requested versions from dependency graph
- By default, selects the highest version
- Version conflicts are resolved automatically unless `failOnVersionConflict()` is enabled

**Configuration options**:
- `failOnVersionConflict()`: Fail build on any conflict
- `force 'group:artifact:version'`: Force specific version
- `preferProjectModules()`: Prefer local project over binary
- `constraints {}`: Suggest versions without requiring

**Trade-offs**:
- **Automatic resolution**: Less manual intervention
- **May upgrade unexpectedly**: Transitive dependency can bump version
- **Flexible**: Many knobs to control behavior
- **Different from Maven**: Can cause confusion when migrating

**References**:
- [Dependency Resolution](https://docs.gradle.org/current/userguide/dependency_resolution.html)
- [Dependency Constraints and Conflict Resolution](https://docs.gradle.org/current/userguide/dependency_constraints_conflicts.html)
- [ResolutionStrategy DSL](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.ResolutionStrategy.html)

---

## PHP

### Composer

Composer uses a SAT solver ported from openSUSE's libzypp.

**Algorithm**: SAT solving (DPLL/CDCL)

**How it works**:
- Translates dependencies into boolean satisfiability clauses
- Uses conflict-driven clause learning (CDCL)
- Finds assignment that satisfies all constraints or proves none exists

**Clause generation**:
- `A requires B`: `(-A|B1|B2|...)`
- `A conflicts with B`: `(-A|-B)`
- Policy rules for updates

**Trade-offs**:
- **Correct**: SAT solvers are well-studied
- **Can be slow**: PHP implementation is not as fast as native SAT solvers
- **NP-complete**: Worst case exponential, but usually fine in practice

**Performance note**: Experiments showed native SAT solver (Plingeling) solving same formula 633x faster than Composer's PHP implementation.

**References**:
- [Composer's SAT Solver (2012 presentation)](https://www.naderman.de/slippy/tmp/2012-06-07-Composers-SAT-Solver.html)
- [Dependency Resolution with SAT](https://www.slideshare.net/naderman/dependency-resolution-with-sat-symfony-live-2011-paris)
- [Solver.php source](https://github.com/composer/composer/blob/main/src/Composer/DependencyResolver/Solver.php)

---

## .NET

### NuGet

NuGet defaults to lowest matching version for dependencies.

**Algorithm**: Lowest applicable version (configurable)

**How it works**:
- When constraint is `>= 2.1`, picks 2.1 if available, or next lowest
- For transitive dependencies, uses lowest version that satisfies constraint
- Floating versions (e.g., `6.0.*`) select highest matching version

**Configuration**:
- `-DependencyVersion` switch: `Lowest` (default), `HighestPatch`, `HighestMinor`, `Highest`
- Only applies to `packages.config` projects, not `PackageReference`

**Rationale**: Lowest version is most likely to be compatible since that's what the package author tested with.

**Trade-offs**:
- **Conservative**: Less likely to break
- **May miss improvements**: Does not get latest patches automatically
- **Different from most managers**: Most others pick highest

**References**:
- [NuGet Package Dependency Resolution](https://learn.microsoft.com/en-us/nuget/concepts/dependency-resolution)
- [Package Versioning](https://learn.microsoft.com/en-us/nuget/concepts/package-versioning)

---

## Dart/Flutter

### pub

pub uses PubGrub, an algorithm designed specifically for Dart.

**Algorithm**: PubGrub (conflict-driven clause learning)

**How it works**:
- Starts with partial solution, iteratively selects packages
- On conflict, derives incompatibility and backtracks
- Uses unit propagation and logical resolution
- Generates human-readable error messages from incompatibility chain

PubGrub was designed to provide clear error messages explaining why resolution failed.

**Trade-offs**:
- **Good error messages**: Explains dependency conflicts in plain English
- **Fast**: Uses modern SAT-solving techniques
- **Widely adopted**: Ported to Rust (pubgrub-rs), used by uv and others

**References**:
- [Dart pub solver documentation](https://github.com/dart-lang/pub/blob/master/doc/solver.md)
- [pubgrub-rs](https://github.com/pubgrub-rs/pubgrub)
- [Package versioning](https://dart.dev/tools/pub/versioning)

---

## Elixir

### Mix/Hex

**Algorithm**: Highest version, single version per package

**How it works**:
- Dependency resolution always tries to use latest version of all packages
- Fails if incompatible version requirements exist
- Single version of each package in the VM (cannot have multiple versions)

**Options**:
- `:override` option forces a dependency version to be used everywhere

**Trade-offs**:
- **Simple model**: Only one version of each package
- **Clear conflicts**: Either resolves or fails with error
- **No multiple versions**: Cannot work around conflicts by having two versions

**References**:
- [Mix usage](https://hex.pm/docs/usage)
- [mix deps](https://hexdocs.pm/mix/1.12/Mix.Tasks.Deps.html)

---

## Haskell

### Cabal

Cabal uses a modular solver with configurable backtracking.

**Algorithm**: Modular solver with backjumping

**How it works**:
- Inspired by Nordin and Tolmach's modular lazy search
- Uses backjumping to skip irrelevant states
- `--reorder-goals` heuristic can speed up some resolutions
- `--count-conflicts` prefers goals involved in many conflicts (default)

**Configuration**:
- `max-backjumps`: Maximum backtrack steps (-1 for unlimited, default 2000)
- `--reorder-goals`: Try to order goals more efficiently
- `--count-conflicts`: Prioritize conflict-heavy packages

**Trade-offs**:
- **Configurable**: Many knobs to tune behavior
- **Can be slow**: Complex dependency graphs cause extensive backtracking
- **Single instance**: Avoids multiple versions of same package in build plan

**References**:
- [cabal.project Reference](https://cabal.readthedocs.io/en/3.4/cabal-project.html)
- [Modular Solver](https://wiki.haskell.org/HaskellImplementorsWorkshop/2011/Loeh)

---

## Swift/iOS

### CocoaPods

CocoaPods uses Molinillo, the same resolver as Bundler.

**Algorithm**: Molinillo (backtracking with forward checking)

See Bundler section for algorithm details.

**References**:
- [Molinillo](https://github.com/CocoaPods/Molinillo)
- [Molinillo Architecture](https://github.com/CocoaPods/Molinillo/blob/master/ARCHITECTURE.md)

---

### Swift Package Manager

Swift Package Manager uses PubGrub for dependency resolution.

**Algorithm**: PubGrub

**How it works**:
- Same PubGrub algorithm used by Dart pub, Poetry, uv
- `Package.resolved` records resolved versions
- Target-based resolution (Swift 5.2+): only resolves dependencies actually needed by included targets

**Trade-offs**:
- **Good error messages**: PubGrub explains why resolution failed
- **Integrated with Xcode**: First-class support in Apple tooling

**References**:
- [PubGrub implementation PR](https://github.com/apple/swift-package-manager/pull/1918)
- [Swift Package Manager source](https://github.com/apple/swift-package-manager/tree/main/Sources/PackageGraph)

---

## System Package Managers

### APT (Debian/Ubuntu)

APT uses a scoring-based resolver with immediate dependency resolution.

**Algorithm**: Scoring with immediate resolution

**How it works**:
- Packages are assigned scores based on importance (Essential: 100, Required: 3, Important: 2, etc.)
- Uses two-stage resolution: first marks packages for action, then resolves
- For OR dependencies, examines alternatives in declared order
- Pre-depends must be installed and configured before dependent package
- dpkg itself does not resolve dependencies; APT handles this layer

**Trade-offs**:
- **Scoring guides choices**: More important packages preferred
- **Order matters for OR**: First satisfying alternative chosen
- **Separate tools**: dpkg for low-level, apt for resolution
- **Mature**: Decades of use on Debian-based systems

**References**:
- [Dependency resolution in aptitude](https://www.debian.org/doc/manuals/aptitude/ch02s03s01.en.html)
- [Immediate dependency resolution](https://www.debian.org/doc/manuals/aptitude/ch02s03s02.en.html)

---

### DNF/YUM (Fedora/RHEL)

DNF uses libsolv, a SAT-based dependency resolver from openSUSE.

**Algorithm**: SAT solving via libsolv

**How it works**:
- Formulates dependencies as a Boolean satisfiability (SAT) problem
- Uses a reimplementation of the Minisat solver
- Hawkey library interfaces between DNF and libsolv
- Parses RPM metadata to construct dependency graph
- Finds minimal set of packages satisfying all constraints

**History**: DNF replaced YUM in Fedora 22+ and RHEL 8+. YUM's ad-hoc dependency checking was slow and unpredictable; libsolv provides modern SAT-based resolution.

**Trade-offs**:
- **Fast**: Native C implementation with optimized SAT solver
- **Correct**: SAT solvers are mathematically well-founded
- **Shared**: Same libsolv used by Zypper (openSUSE)
- **Complex metadata**: RPM repositories have rich dependency information

**References**:
- [libsolv](https://github.com/openSUSE/libsolv)
- [DNF GitHub](https://github.com/rpm-software-management/dnf)
- [Features/DNF](https://fedoraproject.org/wiki/Features/DNF)

---

### Pacman (Arch Linux)

Pacman uses [libalpm](https://gitlab.archlinux.org/pacman/pacman/-/tree/master/lib/libalpm) for package management. Limited public documentation on the resolution algorithm internals.

**How it works**:
- Resolves dependencies during install/update
- Single version of each package system-wide
- Supports optional dependencies (not installed by default)

**Trade-offs**:
- **Simple model**: One version per package
- **Rolling release**: Always latest versions

**References**:
- [pacman ArchWiki](https://wiki.archlinux.org/title/Pacman)

---

### Homebrew (macOS/Linux)

Homebrew has a simpler model than most package managers: one version of each formula at a time.

**Algorithm**: Single version per formula, topological sort for install order

**How it works**:
- Each formula specifies its dependencies (no version ranges)
- Only one version of each formula is available at a time in a given tap
- Dependencies installed in topological order
- Build-time dependencies can be skipped when installing from bottles

**Trade-offs**:
- **No version conflicts**: Only one version exists
- **Upgrade cascades**: Updating a dependency may require rebuilding dependents
- **Simple model**: No constraint solving needed

**References**:
- [Formula Cookbook](https://docs.brew.sh/Formula-Cookbook)

---

### Nix

Nix avoids traditional resolution by having each package explicitly specify exact versions of its dependencies.

**Algorithm**: No resolution needed - dependencies are explicit

**How it works**:
- Each package (derivation) specifies exact dependencies, not version ranges
- No constraint solving or version selection at install time
- Multiple versions of the same package can coexist
- Nixpkgs (the package set) determines which versions are available together

**Trade-offs**:
- **No dependency conflicts**: Each package gets exactly what it declares
- **Multiple versions**: Different packages can use different versions of the same dependency
- **Nixpkgs is the constraint**: Available versions determined by which nixpkgs revision you use

**References**:
- [How Nix Works](https://nixos.org/guides/how-nix-works/)
- [Nix Manual](https://nixos.org/manual/nix/stable/introduction)

---

### Snap

Snaps bundle their dependencies rather than resolving them at install time.

**Algorithm**: No runtime resolution - dependencies bundled at build time

**How it works**:
- Dependencies are bundled into the snap at build time
- Snapcraft uses APT to resolve build dependencies during snap creation
- At install time, no resolution needed since everything is bundled
- Base snaps provide common runtime dependencies

**Trade-offs**:
- **No dependency conflicts at runtime**: Everything bundled
- **Larger package sizes**: Each snap carries its own dependencies
- **Build-time resolution only**: Uses APT for build dependencies

**References**:
- [Manage dependencies](https://snapcraft.io/docs/build-and-staging-dependencies)

---

### Alpine APK

**Algorithm**: Unknown (limited documentation on internals)

**How it works**:
- Maintains "World" file listing explicitly installed packages
- Resolves dependencies to satisfy World requirements
- Supports virtual packages (multiple packages can provide same capability)
- Single version of each package

**References**:
- [Alpine Package Keeper](https://wiki.alpinelinux.org/wiki/Alpine_Package_Keeper)

---

## HPC

### Spack

Spack is a package manager for HPC that uses Answer Set Programming (ASP) for dependency resolution.

**Algorithm**: Answer Set Programming (ASP) via Clingo

**How it works**:
- Uses Clingo, an ASP solver, for "concretization" (resolving abstract specs to concrete versions)
- Models dependencies as logic programming rules and constraints
- Optimizes for user preferences (most tested, most optimized, etc.)
- Can handle complex constraints like compiler versions, build variants, and target architectures

**Why ASP over SAT**: SAT finds any satisfying solution; ASP finds an optimal solution according to user-defined criteria.

**Trade-offs**:
- **Handles HPC complexity**: Compiler versions, MPI implementations, GPU targets
- **Optimization**: Finds best solution, not just any solution
- **Slower than SAT**: More expressive but more expensive

**References**:
- [Concretizer settings](https://spack.readthedocs.io/en/latest/build_settings.html)
- [Spack's new Concretizer (FOSDEM 2020)](https://archive.fosdem.org/2020/schedule/event/dependency_solving_not_just_sat/)

---

## Algorithm Comparison

| Package Manager | Algorithm | Default Version | Lockfile | Multiple Versions |
|-----------------|-----------|-----------------|----------|-------------------|
| npm | Dedup + nesting | Highest | Yes | Yes (nested) |
| Yarn Classic | Dedup + nesting | Highest | Yes | Yes (nested) |
| Yarn Berry | Dedup + nesting | Highest | Yes | Yes (isolated) |
| pnpm | Dedup + nesting | Highest | Yes | Yes (isolated) |
| Bun | Dedup + nesting | Highest | Yes | Yes (nested) |
| pip | Backtracking | Highest | No* | No |
| Poetry | PubGrub | Highest | Yes | No |
| uv | PubGrub | Highest | Yes | No |
| Conda/Mamba | SAT (libsolv) | Highest | Yes | No |
| Bundler | Molinillo | Highest | Yes | No |
| Cargo | Backtracking | Highest | Yes | Yes (major) |
| cpanm/Carton | Latest | Highest | Yes (Carton) | No |
| Go | MVS | Lowest | No | No |
| Maven | Nearest | Nearest | No | No |
| Gradle | Newest | Highest | Yes | No |
| Composer | SAT | Highest | Yes | No |
| NuGet | Lowest | Lowest | Yes | No |
| pub | PubGrub | Highest | Yes | No |
| Mix/Hex | Latest | Highest | Yes | No |
| Cabal | Modular | Highest | Yes | No |
| CocoaPods | Molinillo | Highest | Yes | No |
| SwiftPM | PubGrub | Highest | Yes | No |
| APT | Scoring | Highest | No | No |
| DNF | SAT (libsolv) | Highest | No | No |
| Pacman | Unknown | Latest | No | No |
| Homebrew | Formula-based | Latest | No | No |
| Nix | Explicit (no resolution) | Specified | Yes (flakes) | Yes (by design) |
| Snap | Bundled (no resolution) | N/A | No | N/A |
| Alpine APK | Unknown | Latest | No | No |
| Spack | ASP (Clingo) | Optimized | Yes | Yes |

*pip has no built-in lockfile; `pip freeze > requirements.txt` with pinned versions serves a similar purpose

---

## Common Resolution Strategies

### Highest Version (Most Common)
Select the highest version that satisfies all constraints. Used by most modern package managers.

**Pros**: Gets latest features and security fixes
**Cons**: More likely to introduce breaking changes

### Lowest Version (NuGet default, Go MVS)
Select the lowest version that satisfies all constraints.

**Pros**: More stable, uses what was tested
**Cons**: May miss security patches

### Nearest Definition (Maven)
Select version based on proximity in dependency graph.

**Pros**: Gives control to direct dependencies
**Cons**: Order-dependent, can be surprising

### Deduplication with Nesting (npm, Yarn, pnpm, Bun)
Try to find versions that satisfy multiple dependents; nest different versions when conflicts arise.

**Pros**: Handles conflicts without failing, disk efficient when versions align
**Cons**: Phantom dependencies possible (except pnpm/Yarn PnP), multiple copies when conflicts exist

### Strict Isolation (pnpm, Yarn PnP)
Each package only sees its declared dependencies.

**Pros**: No phantom dependencies
**Cons**: Some packages may break if they rely on hoisted dependencies

---

## Phantom Dependencies

A phantom dependency is when package A can `require('B')` even though A does not list B in its dependencies, simply because B was installed for another package and hoisted to a common ancestor.

**Affected by**: npm, Yarn Classic
**Prevented by**: pnpm, Yarn PnP (strict mode)

This causes problems when:
- The package depending on B is removed
- Different version of B is needed
- Different environment has different hoisting result

---

## Complexity Note

General dependency resolution with arbitrary version constraints is NP-hard (reducible to SAT) when you must select exactly one version of each package. However, not all package managers face this complexity:

- **npm/Yarn/pnpm** attempt to deduplicate by finding versions that satisfy multiple dependents, but can nest different versions when conflicts arise. Peer dependencies complicate this since they must be at the same level and cannot be resolved by nesting.
- **Go's MVS** is polynomial-time because it always picks the minimum version, avoiding the combinatorial explosion of choosing among candidates
- **SAT-based resolvers** (Composer, libsolv) embrace the complexity and use optimized solvers
- **Backtracking resolvers** (pip, Cargo) can hit exponential worst cases but use heuristics to perform well in practice
- **PubGrub** uses conflict-driven clause learning to prune the search space efficiently

---

## License

[CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/)
