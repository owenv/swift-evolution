# Unifying Package Products and Targets

* Proposal: [SE-NNNN](NNNN-unify-package-products-and-targets.md)
* Authors: [Owen Voorhees](https://github.com/owenv)
* Implementation: 
* Review: ([pitch](TBD))

## Introduction

Today, packages distinguish between the concept of targets, which represent a single module, and products, which group one or more targets into a binary other packages can consume. This distinction is often confusing in practice, and makes it difficult to express more complex build configurations. This proposal deprecates the concept of package products and extends the capabilities of package targets to support their use cases.

## Motivation

Today, packages are divided into targets and products. Generally speaking, targets:
- organize source code into a single module
- cannot be directly depended upon by other packages
- do not control they type of image they're linked into
- can specify linker settings

Products, on the other hand:
- organize one or more targets into a library or executable
- can be directly depended upon by other packages
- may express an opinion about how they are linked
- cannot specify linker settings

In practice, this model has been confusing to many users, and the distinction between targets and products has become blurry. For example, an `executableTarget` is used to compile sources for linking into a `executable` product, which is redundant and error prone. A `testTarget` describes how its sources should be built and linked, but can’t be included in a product. Finally, it’s not possible for a package’s target to depend on one of its own products, which can make it more difficult to organize a larger project, and to control linkage when building package products for distribution or installation. This problem will exacerbated if we continue to introduce new product types and postprocessing steps to SwiftPM which have specific packaging or linking requirements. 

Many of these problems are caused by the fact that products currently conflate two unrelated concepts: how code is linked, and whether it's available for dependent packages to consume. By deprecating products in favor of more flexible targets, we can separate these two concepts and allow users to configure them independently.

## Proposed solution

Deprecate the concept of package products and related manifest API. In its place, introduce new target types to represent static, dynamic, and "automatic" library targets. Also introduce a new `exported` parameter allowing targets to specify a target in a dependent package may consume them.

## Detailed design

### New Target Capabilities

Today, there are 3 types of products a package can declare: libraries, executables, and plugins.

Library products will be replaced by 3 new target types which represent dynamic, static, and "automatic" libraries:

```swift
    @available(_PackageDescription, introduced: 999.0)
    public static func dynamicLibraryTarget(
        name: String,
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
        exported: Bool = false
    ) -> Target

    @available(_PackageDescription, introduced: 999.0)
    public static func staticArchiveTarget(
        name: String,
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
        exported: Bool = false
    ) -> Target

    @available(_PackageDescription, introduced: 999.0)
    public static func automaticLibraryTarget(
        name: String,
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
        exported: Bool = false
    ) -> Target
```

A `dynamicLibraryTarget` will produce a dynamic library, and a `staticArchiveTarget` will produce a static archive. An `automaticLibraryTarget` leaves the decision to the build system as a `library` product with no specified linkage does today. All three new target types are permitted to omit the sources directory as long as they have at least one dependency on a regular `target`. This allows the new target types to either aggregate multiple targets into a single linked library, or include sources themselves and represent the contents of a single module.

The existing `executableTarget` replaces an `executable` product and continues to produce an executable without requiring any specific changes to the manifest API. Similarly, the existing `plugin` target API replaces the corresponding product API.

The new `dynamicLibraryTarget`, `staticArchiveTarget`, and `automaticLibraryTarget` API, along with the existing `target`, `executableTarget`, `binaryTarget`, `plugin`, and `systemLibrary` API each gain a new parameter:

```swift
exported: Bool = false
```

`testTarget`s may not be exported as they can't be dependend upon by other targets.

If a target specifies that it's `exported`, a target in another package can declare a dependency on it using new `Target.Dependency` API:
```swift
    @available(_PackageDescription, introduced: 5.7)
    public static func target(
      name: String,
      package: String,
      moduleAliases: [String: String]? = nil,
      condition: TargetDependencyCondition? = nil
    ) -> Target.Dependency
```

If a package declares a dependency on a target from another package that isn't exported, SwiftPM will report an error.

### Deprecating Products

The existing `Package` initializer which accepts a `products` array will be deprecated in favor of a new one where it's removed:

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

Although the `products` parameter is deprecated, SwiftPM will continue to respect it if present, to ease the migration of existing packages. Additionally, the `product` methods which construct a `Target.Dependency` are _not_ deprecated by this proposal. This means:
- An existing package may adopt the new tools version before fully migrating its products to exported targets, although a deprecation warning will be emitted in the manifest
- After adopting the new tools version, a package may continue to depend on products from other packages, including those with the latest or an older tools version

## Security

This proposal only impacts how packages organize and build their sources. It does not meaningfully impact package manager security.

## Impact on existing packages

- Packages using existing tools versions will continue to build as they do today.
- When upgrading their tools-version, manifests with a `products` list will report a deprecation warning but otherwise continue to build as they do today. However, it's recommended they migrate their products to exported targets.

## Alternatives considered

### Continue to Evolve Targets and Products Independently

This is a large and disruptive change to how package manifests are authored. While the change significantly simplifies the manifest API and solves issues like the current inability to declare dependencies between products in the same package, it could be argued that we should instead pursue more targeted changes to the existing product/target split.

### Immediately Obsolete Product Related Manifest API

Instead of deprecating products, but continuing to support declaring them as a transitional aid, we could immediately obsolete the related API in the upcoming tools version. This alternative was rejected as being too disruptive to existing packages which might want to adopt new tools versions without immediately taking on a major migration.

## Acknowledgements



TODO:
- Any CLI options need edits?
- Find all the places we assume a target -> product dependency edge must be to another package
- Plugin API changes?
- REPL product changes