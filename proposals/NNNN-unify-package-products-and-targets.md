# Unifying Package Products and Targets

* Proposal: [SE-NNNN](NNNN-unify-package-products-and-targets.md)
* Authors: [Owen Voorhees](https://github.com/owenv)
* Implementation: [PR-10500 (WIP)](https://github.com/swiftlang/swift-package-manager/pull/10500)
* Review: ([pitch](TBD))

## Introduction

Today, packages distinguish between the concept of targets, which represent a single module, and products, which group one or more targets into a binary other packages can consume. This distinction is often confusing in practice, and makes it difficult to express more complex builds. This proposal deprecates the concept of package products and extends the capabilities of package targets to support existing use cases and enable new ones.

## Motivation

Today, packages are divided into targets and products. Generally speaking, targets:
- organize source code into a single module
- cannot be directly depended upon by other packages
- do not control the type of image they're linked into
- can specify linker settings

Products, on the other hand:
- organize one or more targets into a library or executable
- can be directly depended upon by other packages
- may express an opinion about how they are linked
- cannot specify linker settings

In practice, this model has been confusing to many users, and the distinction between targets and products has become blurry. For example, an `executableTarget` is used to compile sources for linking into a `executable` product, which is redundant and easy to forget. A `testTarget` describes how its sources should be built and linked, but can’t be included in a product. The lack of consistent rules makes it difficult to build a mental model of how packages are organized from first principles.

The split between products and targets also limits the expressivity of packages. A longstanding limitation of SwiftPM is that it's not possible for one of a package’s targets to depend on one of its own products. This makes organizing larger packages difficult, and limits users' control over how code within a package is linked. Users must resort to awkward workarounds like breaking their code into multiple packages or integrating with external build systems to regain control. This problem will be exacerbated as SwiftPM continues evolving to support building increasingly complex software with new kinds of targets and products.

Many of these issues are caused by the fact that products currently conflate two unrelated concepts: how code is linked, and whether it's available for dependent packages to consume. By deprecating products in favor of more flexible targets, we can separate these two concepts and allow users to configure them independently.

## Proposed solution

Deprecate the concept of package products and related manifest API. In its place, introduce new target types to represent static, dynamic, and "automatic" library targets. Also introduce a new `visibility` parameter allowing targets to control whether a target in a dependent package may consume them.

## Detailed design

### New Target Types

Today, there are 3 types of products a package can declare: libraries, executables, and plugins.

Library products will be replaced by a new target type, `libraryTarget`:

```swift
    @available(_PackageDescription, introduced: 999.0)
    public static func libraryTarget(
        name: String,
        type: LibraryType? = nil,
        dependencies: [Dependency] = [],
        path: String? = nil,
        exclude: [String] = [],
        sources: [String]? = nil,
        resources: [Resource]? = nil,
        publicHeadersPath: String? = nil,
        packageAccess: Bool = true,
        cSettings: [CSetting]? = nil,
        cxxSettings: [CXXSetting]? = nil,
        swiftSettings: [SwiftSetting]? = nil,
        linkerSettings: [LinkerSetting]? = nil,
        plugins: [PluginUsage]? = nil,
        visibility: TargetVisibility = .package
    ) -> Target
```

`Product.Library.LibraryType` will become a `typealias` of the new top-level `LibraryType` enum.

A `libraryTarget` with `type: .dynamic` will produce a dynamic library, and a `libraryTarget` with `type: .static` will produce a static library. A `libraryTarget` with no `type` leaves the decision to the build system just as a `library` product with no specified linkage does today.

A `libraryTarget` is permitted to omit the sources directory as long as it has at least one dependency on a regular `target`. This allows the new target type to either aggregate multiple targets into a single library, or include sources itself and represent the contents of a single module. As a result, it can serve as a replacement for all current uses of library products. A `libraryTarget` with no sources of its own does not define a module.

The existing `executableTarget` replaces an `executable` product and continues to produce an executable without requiring any specific changes to the manifest API. Similarly, the existing `plugin` target API replaces the corresponding product API.

### Target Visibility

Instead of exposing functionality to dependent packages through products, package authors will now do so by specifying a target's visibility. The new `libraryTarget` API, along with the existing `target`, `executableTarget`, `testTarget`, `binaryTarget`, `plugin`, `macro`, and `systemLibrary` API each gain a new parameter:

```swift
visibility: TargetVisibility = .package
```

```swift
 @available(_PackageDescription, introduced: 999.0)
 public enum TargetVisibility {
    case `public`
    case `package`
 }
```

If a target has `.public` visibility, a target in the same package or a dependent package may declare a dependency on it. If a target has `.package` visibility, only targets in the same package are allowed to declare a dependency on it. If visibility isn't specified explicitly, targets default to `.package` visibility to ensure package authors don't accidentally expose API they didn't intend to.

If a target specifies that it has `visibility: .public`, a target in another package can declare a dependency on it using new `Target.Dependency` API:
```swift
    @available(_PackageDescription, introduced: 999.0)
    public static func target(
      name: String,
      package: String,
      moduleAliases: [String: String]? = nil,
      condition: TargetDependencyCondition? = nil
    ) -> Target.Dependency
```

Like products, this new API allows specifying module aliases when depending on a target in another package. Specifying module aliases in a dependency on a target from the same package remains disallowed.

### Deprecating Products

The existing `Package` initializer which accepts a `products` array will be deprecated (but not obsoleted) in favor of a new one where it's removed:

```swift
    @available(_PackageDescription, introduced: 999.0)
    public init(
        name: String,
        defaultLocalization: LanguageTag? = nil,
        platforms: [SupportedPlatform]? = nil,
        pkgConfig: String? = nil,
        providers: [SystemPackageProvider]? = nil,
        traits: Set<Trait> = [],
        dependencies: [Dependency] = [],
        targets: [Target] = [],
        swiftLanguageModes: [SwiftLanguageMode]? = nil,
        cLanguageStandard: CLanguageStandard? = nil,
        cxxLanguageStandard: CXXLanguageStandard? = nil
    )
```

Although the `products` parameter is deprecated, SwiftPM will continue to respect it if present. This means an existing package may adopt the new tools version before fully migrating its products to public targets, to ease the migration process. Additionally, the `product` methods which construct a `Target.Dependency` are _not_ deprecated by this proposal. After adopting the new tools version, a package may continue to depend on products declared by other packages, including packages with older tools versions. Finally, if a `product` dependency finds no products with the given name in the specified package, it will fall back to searching for a target with the specified name and public visibility. This allows a package to migrate from products to public targets without breaking clients.

SwiftPM will report an error if a product attempts to include a `libraryTarget`, and the general expectation is that products will not be able to aggregate any new target types introduced in the future.

### CLI Changes

In general, existing CLI options for working with products will continue to be supported to allow working with packages that have older tools versions. For example, both `--target` and `--product` will continue to be supported by `swift build`.

However, various assorted changes to the SwiftPM CLI will be made to accommodate the new capabilities of targets:
- `swift package --generate-sbom` will gain a `--target` flag. Similar to the existing `--product` flag, it will generate an SBOM for a specific target.
- `swift package add-target` currently allows specifying `--type library` to add a regular `target`. When a manifest uses the new tools version, `--type library` will now insert a `libraryTarget` with no type, `--type dynamic-library` will insert one with `type: .dynamic`, and `--type static-library` will insert one with `type: .static`. `--type none` or no `--type` flag will insert a regular `target`. The subcommand will also now accept `--visibility public` or `--visibility package` as optional arguments.
- `swift package describe` output will be updated with a new field for target visibility.
- Currently when a command plugin is invoked with `swift package my-command-plugin`, SwiftPM considers plugins in the root package, and plugin products in direct dependencies. It will be updated to also consider plugin targets with `visibility: .public` in direct dependencies.
- Currently, `swift run --repl` exposes all library products in the root package to the REPL. It will be updated to also include any `libraryTarget` with `visibility: .public`.
- Currently, `swift package diagnose-api-breaking-changes` defaults to comparing the API of all modules incorporated in library products. It will be updated to also include any `libraryTarget` with `visibility: .public`.

### PackagePlugin API Changes

For the most part, the PackagePlugin API adapts straightforwardly to the new model:
- `ModuleKind` will add a new `library` case.
- A `libraryTarget` with no sources will be represented using a new internal type which conforms to the existing `Target` protocol.
- `Target` will add a new `visibility` field:
```swift
@available(_PackageDescription, introduced: 999.0)
var visibility: TargetVisibility { get }
```

### Example: swift-driver and swift-argument-parser

swift-driver is a package which implements Swift's compiler driver, a component of the Swift toolchain. It's a medium sized package which demonstrates some of the benefits of this proposal. Currently, its manifest looks like this (with some slight simplifications):
```swift
// swift-tools-version:5.10

import PackageDescription

let package = Package(
  name: "swift-driver",
  platforms: [
    .macOS(.v12),
    .iOS(.v15),
  ],
  products: [
    .executable(
      name: "swift-driver",
      targets: ["swift-driver"]),
    .executable(
      name: "swift-help",
      targets: ["swift-help"]),
    .executable(
      name: "swift-build-sdk-interfaces",
      targets: ["swift-build-sdk-interfaces"]),
    .library(
      name: "SwiftDriver",
      targets: ["SwiftDriver"]),
    .library(
      name: "SwiftDriverDynamic",
      type: .dynamic,
      targets: ["SwiftDriver"]),
    .library(
      name: "SwiftOptions",
      targets: ["SwiftOptions"]),
    .library(
      name: "SwiftDriverExecution",
      targets: ["SwiftDriverExecution"]),
  ],
  dependencies: [
    .package(url: "https://github.com/swiftlang/swift-tools-support-core.git", branch: "main"),
    .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.1"),
    .package(url: "https://github.com/swiftlang/swift-llbuild.git", branch: "main"),
  ],
  targets: [
    /// C modules wrapper for _InternalLibSwiftScan.
    .target(name: "CSwiftScan",
            exclude: [ "CMakeLists.txt" ]),

    /// The driver library.
    .target(
      name: "SwiftDriver",
      dependencies: [
        "SwiftOptions",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
        "CSwiftScan",
      ],
      exclude: ["CMakeLists.txt"]),

    /// The execution library.
    .target(
      name: "SwiftDriverExecution",
      dependencies: [
        "SwiftDriver",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core")
      ],
      exclude: ["CMakeLists.txt"]),

    /// Driver tests.
    .testTarget(
      name: "SwiftDriverTests",
      dependencies: ["SwiftDriver", "SwiftDriverExecution", "TestUtilities", "ToolingTestShim"]),

    /// IncrementalImport tests
    .testTarget(
      name: "IncrementalImportTests",
      dependencies: [
        "IncrementalTestFramework",
        "TestUtilities",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ]),

    .target(
      name: "IncrementalTestFramework",
      dependencies: [ "SwiftDriver", "SwiftOptions", "TestUtilities" ],
      path: "Tests/IncrementalTestFramework"),

    .target(
      name: "TestUtilities",
      dependencies: ["SwiftDriver", "SwiftDriverExecution"],
      path: "Tests/TestUtilities"),

    .target(
      name: "ToolingTestShim",
      dependencies: ["SwiftDriver"],
      path: "Tests/ToolingTestShim"),

    /// The options library.
    .target(
      name: "SwiftOptions",
      dependencies: [
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"]),
    .testTarget(
      name: "SwiftOptionsTests",
      dependencies: ["SwiftOptions"]),

    /// The primary driver executable.
    .executableTarget(
      name: "swift-driver",
      dependencies: ["SwiftDriverExecution", "SwiftDriver"],
      exclude: ["CMakeLists.txt"]),

    /// The help executable.
    .executableTarget(
      name: "swift-help",
      dependencies: [
        "SwiftOptions",
        .product(name: "ArgumentParser", package: "swift-argument-parser"),
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"]),

    /// Build SDK Interfaces tool executable.
    .executableTarget(
      name: "swift-build-sdk-interfaces",
      dependencies: ["SwiftDriver", "SwiftDriverExecution"],
      exclude: ["CMakeLists.txt"]),

    /// The `makeOptions` utility (for importing option definitions).
    .executableTarget(
      name: "makeOptions",
      dependencies: [],
      // Do not enforce checks for LLVM's ABI-breaking build settings.
      // makeOptions runtime uses some header-only code from LLVM's ADT classes,
      // but we do not want to link libSupport into the executable.
      cxxSettings: [.unsafeFlags(["-DLLVM_DISABLE_ABI_BREAKING_CHECKS_ENFORCING=1"])],
      linkerSettings: [
        .linkedLibrary("swiftCore", .when(platforms: [.windows])), // for swift_addNewDSOImage
      ]),
  ],
  cxxLanguageStandard: .cxx17
)
```

Immediately you'll notice that the list of products is largely redundant. Aside from "SwiftDriverDynamic", which configures linkage, all the products simply expose a single target to dependents. With the changes in this proposal, the manifest can be simplified to the following:

```swift
// swift-tools-version:999.0

import PackageDescription

let package = Package(
  name: "swift-driver",
  platforms: [
    .macOS(.v12),
    .iOS(.v15),
  ],
  dependencies: [
    .package(url: "https://github.com/swiftlang/swift-tools-support-core.git", branch: "main"),
    .package(url: "https://github.com/apple/swift-argument-parser.git", from: "1.5.1"),
    .package(url: "https://github.com/swiftlang/swift-llbuild.git", branch: "main"),
  ],
  targets: [
    /// C modules wrapper for _InternalLibSwiftScan.
    .target(name: "CSwiftScan",
            exclude: [ "CMakeLists.txt" ]),

    /// The driver library.
    .libraryTarget(
      name: "SwiftDriver",
      type: .dynamic,
      dependencies: [
        "SwiftOptions",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
        "CSwiftScan",
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),

    /// The execution library.
    .libraryTarget(
      name: "SwiftDriverExecution",
      dependencies: [
        "SwiftDriver",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core")
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),

    /// Driver tests.
    .testTarget(
      name: "SwiftDriverTests",
      dependencies: ["SwiftDriver", "SwiftDriverExecution", "TestUtilities", "ToolingTestShim"]),

    /// IncrementalImport tests
    .testTarget(
      name: "IncrementalImportTests",
      dependencies: [
        "IncrementalTestFramework",
        "TestUtilities",
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ]),

    .target(
      name: "IncrementalTestFramework",
      dependencies: [ "SwiftDriver", "SwiftOptions", "TestUtilities" ],
      path: "Tests/IncrementalTestFramework"),

    .target(
      name: "TestUtilities",
      dependencies: ["SwiftDriver", "SwiftDriverExecution"],
      path: "Tests/TestUtilities"),

    .target(
      name: "ToolingTestShim",
      dependencies: ["SwiftDriver"],
      path: "Tests/ToolingTestShim"),

    /// The options library.
    .libraryTarget(
      name: "SwiftOptions",
      dependencies: [
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),
    .testTarget(
      name: "SwiftOptionsTests",
      dependencies: ["SwiftOptions"]),

    /// The primary driver executable.
    .executableTarget(
      name: "swift-driver",
      dependencies: ["SwiftDriverExecution", "SwiftDriver"],
      exclude: ["CMakeLists.txt"],
      visibility: .public),

    /// The help executable.
    .executableTarget(
      name: "swift-help",
      dependencies: [
        "SwiftOptions",
        .product(name: "ArgumentParser", package: "swift-argument-parser"),
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),

    /// Build SDK Interfaces tool executable.
    .executableTarget(
      name: "swift-build-sdk-interfaces",
      dependencies: ["SwiftDriver", "SwiftDriverExecution"],
      exclude: ["CMakeLists.txt"],
      visibility: .public),

    /// The `makeOptions` utility (for importing option definitions).
    .executableTarget(
      name: "makeOptions",
      dependencies: [],
      // Do not enforce checks for LLVM's ABI-breaking build settings.
      // makeOptions runtime uses some header-only code from LLVM's ADT classes,
      // but we do not want to link libSupport into the executable.
      cxxSettings: [.unsafeFlags(["-DLLVM_DISABLE_ABI_BREAKING_CHECKS_ENFORCING=1"])],
      linkerSettings: [
        .linkedLibrary("swiftCore", .when(platforms: [.windows])), // for swift_addNewDSOImage
      ]),
  ],
  swiftLanguageModes: [.v5],
  cxxLanguageStandard: .cxx17
)
```

In addition to the ergonomic improvements, swift-driver can also take advantage of the increased expressivity of the manifest. When shipped as part of the toolchain, we want to distribute both the swift-driver executable and the libSwiftDriver shared library which implements much of its functionality. Previously, the swift-driver executable target wasn't able to depend on the SwiftDriverDynamic product which specified dynamic linkage. Now, the SwiftDriver library target can directly specify dynamic linkage, and we're able to build an executable which links a shared library from the same package.

Note that the above manifest still uses product dependencies to include code from swift-llbuild, swift-argument-parser, and swift-tools-support-core. These packages haven't been updated to the new target based model yet, but swift-driver is able to continue using them, allowing the package graph to migrate to the new format one package at a time. Let's say I wanted to migrate swift-argument-parser next. Its manifest currently looks like this:

```swift
import PackageDescription

var package = Package(
  name: "swift-argument-parser",
  products: [
    .library(
      name: "ArgumentParser",
      targets: ["ArgumentParser"]),
    .plugin(
      name: "GenerateDoccReference",
      targets: ["GenerateDoccReference"]),
    .plugin(
      name: "GenerateManual",
      targets: ["GenerateManual"]),
  ],
  dependencies: [],
  targets: [
    // Core Library
    .target(
      name: "ArgumentParser",
      dependencies: ["ArgumentParserToolInfo"],
      exclude: ["CMakeLists.txt"]),
    .target(
      name: "ArgumentParserTestHelpers",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      exclude: ["CMakeLists.txt"]),
    .target(
      name: "ArgumentParserToolInfo",
      exclude: ["CMakeLists.txt"]),

    // Plugins
    .plugin(
      name: "GenerateDoccReference",
      capability: .command(
        intent: .custom(
          verb: "generate-docc-reference",
          description:
            "Generate a documentation reference for a specified target."),
        permissions: [
          .writeToPackageDirectory(
            reason: "This command generates documentation.")
        ]),
      dependencies: ["generate-docc-reference"]),
    .plugin(
      name: "GenerateManual",
      capability: .command(
        intent: .custom(
          verb: "generate-manual",
          description: "Generate a manual entry for a specified target.")),
      dependencies: ["generate-manual"]),

    // Examples
    .executableTarget(
      name: "roll",
      dependencies: ["ArgumentParser"],
      path: "Examples/roll"),
    .executableTarget(
      name: "math",
      dependencies: ["ArgumentParser"],
      path: "Examples/math"),
    .executableTarget(
      name: "repeat",
      dependencies: ["ArgumentParser"],
      path: "Examples/repeat"),
    .executableTarget(
      name: "color",
      dependencies: ["ArgumentParser"],
      path: "Examples/color"),
    .executableTarget(
      name: "default-as-flag",
      dependencies: ["ArgumentParser"],
      path: "Examples/default-as-flag"
    ),

    // Tools
    .executableTarget(
      name: "generate-docc-reference",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      path: "Tools/generate-docc-reference"),
    .executableTarget(
      name: "generate-manual",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      path: "Tools/generate-manual"),

    // Tests
    .testTarget(
      name: "ArgumentParserEndToEndTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt"]),
    .testTarget(
      name: "ArgumentParserExampleTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"],
      resources: [.copy("CountLinesTest.txt")]),
    .testTarget(
      name: "ArgumentParserGenerateDoccReferenceTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"]),
    .testTarget(
      name: "ArgumentParserGenerateManualTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"]),
    .testTarget(
      name: "ArgumentParserPackageManagerTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt"]),
    .testTarget(
      name: "ArgumentParserToolInfoTests",
      dependencies: ["ArgumentParserToolInfo"],
      exclude: ["Examples"]),
    .testTarget(
      name: "ArgumentParserUnitTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt", "Snapshots"]),
  ]
)
```

To migrate it to the new format, we remove the products list and expose the relevant targets with public visibility. The public target names match the old product names, so this change doesn't break the build of swift-driver:

```swift
import PackageDescription

var package = Package(
  name: "swift-argument-parser",
  dependencies: [],
  targets: [
    // Core Library
    .libraryTarget(
      name: "ArgumentParser",
      dependencies: ["ArgumentParserToolInfo"],
      exclude: ["CMakeLists.txt"],
      visibility: .public),
    .target(
      name: "ArgumentParserTestHelpers",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      exclude: ["CMakeLists.txt"]),
    .target(
      name: "ArgumentParserToolInfo",
      exclude: ["CMakeLists.txt"]),

    // Plugins
    .plugin(
      name: "GenerateDoccReference",
      capability: .command(
        intent: .custom(
          verb: "generate-docc-reference",
          description:
            "Generate a documentation reference for a specified target."),
        permissions: [
          .writeToPackageDirectory(
            reason: "This command generates documentation.")
        ]),
      dependencies: ["generate-docc-reference"],
      visibility: .public),
    .plugin(
      name: "GenerateManual",
      capability: .command(
        intent: .custom(
          verb: "generate-manual",
          description: "Generate a manual entry for a specified target.")),
      dependencies: ["generate-manual"]
      visibility: .public),

    // Examples
    .executableTarget(
      name: "roll",
      dependencies: ["ArgumentParser"],
      path: "Examples/roll"),
    .executableTarget(
      name: "math",
      dependencies: ["ArgumentParser"],
      path: "Examples/math"),
    .executableTarget(
      name: "repeat",
      dependencies: ["ArgumentParser"],
      path: "Examples/repeat"),
    .executableTarget(
      name: "color",
      dependencies: ["ArgumentParser"],
      path: "Examples/color"),
    .executableTarget(
      name: "default-as-flag",
      dependencies: ["ArgumentParser"],
      path: "Examples/default-as-flag"
    ),

    // Tools
    .executableTarget(
      name: "generate-docc-reference",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      path: "Tools/generate-docc-reference"),
    .executableTarget(
      name: "generate-manual",
      dependencies: ["ArgumentParser", "ArgumentParserToolInfo"],
      path: "Tools/generate-manual"),

    // Tests
    .testTarget(
      name: "ArgumentParserEndToEndTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt"]),
    .testTarget(
      name: "ArgumentParserExampleTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"],
      resources: [.copy("CountLinesTest.txt")]),
    .testTarget(
      name: "ArgumentParserGenerateDoccReferenceTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"]),
    .testTarget(
      name: "ArgumentParserGenerateManualTests",
      dependencies: ["ArgumentParserTestHelpers"],
      exclude: ["Snapshots"]),
    .testTarget(
      name: "ArgumentParserPackageManagerTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt"]),
    .testTarget(
      name: "ArgumentParserToolInfoTests",
      dependencies: ["ArgumentParserToolInfo"],
      exclude: ["Examples"]),
    .testTarget(
      name: "ArgumentParserUnitTests",
      dependencies: ["ArgumentParser", "ArgumentParserTestHelpers"],
      exclude: ["CMakeLists.txt", "Snapshots"]),
  ]
)
```

And finally, swift-driver can update its dependency declarations on its own schedule from:

```swift
.executableTarget(
      name: "swift-help",
      dependencies: [
        "SwiftOptions",
        .product(name: "ArgumentParser", package: "swift-argument-parser"),
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),
```

to:

```swift
.executableTarget(
      name: "swift-help",
      dependencies: [
        "SwiftOptions",
        .target(name: "ArgumentParser", package: "swift-argument-parser"),
        .product(name: swiftToolsSupportCoreLibName, package: "swift-tools-support-core"),
      ],
      exclude: ["CMakeLists.txt"],
      visibility: .public),
```

Now, swift-driver uses a mix of targets and products from other packages, and package authors can continue to incrementally modernize the remaining packages.

## Security

This proposal only impacts how packages organize and build their sources. It does not impact package manager security.

## Impact on existing packages

- Packages using existing tools versions will continue to build as they do today.
- When upgrading their tools-version, manifests with a `products` list will report a deprecation warning but otherwise continue to build as they do today. However, it's recommended they migrate their products to public targets.

## Alternatives considered

### Continue to Evolve Targets and Products Independently

This proposal is a large and disruptive change to how package manifests are authored. While the change significantly simplifies the manifest API and solves important issues like the current inability to declare dependencies between products in the same package, it could be argued that we should instead pursue more targeted changes to the existing product/target split.

### Immediately Obsolete Product Related Manifest API

Instead of deprecating products, but continuing to support declaring them as a transitional aid, we could immediately obsolete the related API in the upcoming tools version. This alternative was rejected as being too disruptive to existing packages which might want to adopt new tools versions without immediately taking on a major migration.

### Rename Target API

Arguably the meaning of a bare `.target` would be clearer in the new model if it was renamed to something like `.libraryTarget` with `type: .object`. Similarly, it's worth considering if it makes sense to rename API like `macro` and `plugin` to `macroTarget` and `pluginTarget`. For the most part this proposal avoids renaming manifest API whose behavior hasn't meaningfully changed so that the overall change is less disruptive to users.

## Future Directions

### Fine-grained dependency specifications

This proposal gives package authors more control over how individual targets in a package are linked, but not over how they link their dependencies. Today, the meaning of a dependency declared in a package is driven by heuristics which take into account things like the target types involved and the transitive dependencies of a target. Usually, a target dependency means the dependent compiles against and links the dependent's output. This isn’t always the case though. For example, a build tool plugin depending on an executable depends on the executable produced, but the dependency doesn’t imply linkage or interface imports. As packages get more complex, these heuristics won’t always be correct. We should consider introducing new manifest API which allows a dependency to specify whether it’s a compilation, linkage, or runtime dependency (or usually, some combination of the above). Simple packages can continue to use the existing heuristics, but more complex packages will have a new tool to ensure their dependencies are correctly tracked. For now, this proposal defers the design of this feature to future work.

## Acknowledgements

Thanks to Bri Peticca, Joannis Orlandos, Sam Khouri, Sven Schmidt, Robert Connell, Doug Schaefer, Tracy Miranda, and Evan Wilde for their early feedback and suggestions on this proposal.

