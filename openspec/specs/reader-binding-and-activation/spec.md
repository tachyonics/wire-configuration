# Reader binding and activation

## Purpose

The producer Wire synthesises for a `@ConfigProperty` site depends on a `ConfigReader` binding
like any other dependency. This spec covers where that reader comes from, how a site selects one
reader among several, and what a consuming package must declare for the annotation to take effect.

Documentation: [README](../../../README.md).

## Requirements

### Requirement: The reader is an ordinary binding
The synthesised producer SHALL depend on a `ConfigReader` binding resolved by Wire's normal rules.
The package SHALL NOT provide a reader itself; a graph with a `@ConfigProperty` site and no
`ConfigReader` binding SHALL fail at build time with Wire's missing-binding diagnostic.

#### Scenario: reader supplied as a graph input
- **WHEN** a `@GraphInputs` struct declares `let config: ConfigReader` and the app calls `Wire.bootstrap(inputs:)`
- **THEN** every unselected `@ConfigProperty` site reads from that reader

#### Scenario: reader supplied by a producer
- **WHEN** a module declares `@Provides let configReader = ConfigReader(providers: [EnvironmentVariablesProvider()])`
- **THEN** every unselected `@ConfigProperty` site reads from that reader

#### Scenario: no reader bound
- **WHEN** a module has a `@ConfigProperty` site and binds no `ConfigReader`
- **THEN** WireGen reports that no binding produces `ConfigReader` and emits nothing

Pinned by: nothing yet.

### Requirement: A site may select a keyed reader
`@ConfigProperty` SHALL accept `reader: BindingKey<ConfigReader>?` on every form, defaulting to
`nil`. When written as a reference to a declared `BindingKey<ConfigReader>` (for example
`Keys.overrides`), WireGen SHALL key the synthesised producer's `ConfigReader` dependency by that key
and SHALL NOT pass the argument to `wireValue`. When omitted, the reader SHALL resolve by type.
No `wireValue` overload SHALL take a reader key. WireGen reads the argument as source text, so
writing `reader: nil` explicitly does not resolve by type, which is tracked as a defect in
https://github.com/swift-wire/swift-wire/issues/442.

#### Scenario: both spellings at one parameter site
- **WHEN** one function declares two `Int` parameters annotated `@ConfigProperty(forKey: "port", default: 8080)`, one with `reader:` omitted and the other with `reader: Keys.overrides`, and is called with a different value for each
- **THEN** both annotations type-check against the same initialiser and each parameter carries the value passed for it

#### Scenario: two sites with one key and two readers
- **WHEN** two sites read `forKey: "couchdb.port"`, one from the type-resolved reader and one from `Keys.overrides`
- **THEN** the graph holds two distinct bindings, one per selected reader

#### Scenario: an explicit `nil` selector
- **WHEN** a site is written `@ConfigProperty(reader: nil, forKey: "port", default: 8080) port: Int`
- **THEN** WireGen keys the producer's `ConfigReader` dependency by the text `nil` and reports "key 'nil' is referenced but never declared"

Pinned by: `Tests/WireConfigurationTests/ConfigPropertyTests.swift` (`theSelectorIsAcceptedAndDoesNotReachResolution`) for the first scenario. The second is pinned by nothing yet in this repository; the rule is pinned in swift-wire by [InjectionRewriteTests.swift](https://github.com/swift-wire/swift-wire/blob/main/Tests/WireGenCoreTests/InjectionRewriteTests.swift) (`sameArgumentsFromDifferentProvidersAreDistinctBindings`), using two keyed readers. The third is pinned by nothing yet.

### Requirement: Identical sites share one producer
WireGen SHALL synthesise one producer per distinct combination of annotation arguments, site type
and selected reader, so two sites written identically resolve to one binding.

#### Scenario: the same key at two sites
- **WHEN** two `@Provides` functions each declare `@ConfigProperty(forKey: "couchdb.host", default: "localhost") host: String`
- **THEN** the generated graph holds one `String` binding for that key and both functions receive it

Pinned by: nothing yet in this repository. The deduplication rule is specified in swift-wire's adapter-annotations spec.

### Requirement: Consumers depend on the package directly
For `@ConfigProperty` to be recognised, the composition root (the target that applies a Wire build
plugin) SHALL list `WireConfiguration` as a direct dependency alongside `Wire`. This holds even when
the `@ConfigProperty` sites live in a contributor library, which applies no plugin itself. Any build
plugin that runs `WireGen` SHALL suffice, including another adapter's.

#### Scenario: transitive dependency only
- **WHEN** a composition root depends on a library that depends on `WireConfiguration`, but does not list `WireConfiguration` itself
- **THEN** its `@ConfigProperty` sites, and those of any library it composes, are not rewritten, because Wire activates only direct dependencies

#### Scenario: another adapter's plugin
- **WHEN** a target applies wire-mvc's `WireMVCBuildPlugin` rather than `WireBuildPlugin`
- **THEN** its `@ConfigProperty` sites are rewritten, because that plugin runs `WireGen`

Pinned by: nothing yet. The activation rule is specified in swift-wire's multi-module-composition spec.

### Requirement: The package builds on Linux and on macOS with a current SDK
The package SHALL declare a macOS 15 deployment target and SHALL build and test on Linux with
Swift 6.3.3 and 6.4.0. On macOS it SHALL be built against an Xcode 26 SDK. Against an older SDK the
build fails inside swift-configuration
(https://github.com/apple/swift-configuration/issues/178), not in this package.

#### Scenario: the CI matrix
- **WHEN** CI runs
- **THEN** the Linux jobs run on `ubuntu-24.04` with Swift 6.3.3 and 6.4.0, and the macOS jobs run on `macos-26`

Pinned by: `.github/workflows/build.yml` (jobs `Linux / Swift`, `macOS 26 / Swift`) for the builds CI runs. The macOS 15 deployment target and the older-SDK failure are pinned by nothing yet: no job builds against an older SDK.

## Related specifications

- [config-property](../config-property/spec.md)
- [swift-wire multi-module-composition](https://github.com/swift-wire/swift-wire/blob/main/openspec/specs/multi-module-composition/spec.md)
- [swift-wire adapter-annotations](https://github.com/swift-wire/swift-wire/blob/main/openspec/specs/adapter-annotations/spec.md)
- [swift-wire graph-inputs](https://github.com/swift-wire/swift-wire/blob/main/openspec/specs/graph-inputs/spec.md)
