# Config property

## Purpose

`@ConfigProperty` reads a value from a `ConfigReader` at an injection site, so the value itself
becomes the binding a consumer depends on. This spec covers the annotation's three forms, the value
types it supports, the two attachment sites it accepts, and the static read entry points that Wire's
synthesised producer calls.

Documentation: [README](../../../README.md).

## Requirements

### Requirement: The form is decided by the site's type and the presence of a default
`@ConfigProperty` SHALL offer three forms per supported type, told apart only by the annotated
site's type and whether a `default:` argument is written: a defaulted form for a non-optional type
with `default:`, an optional form for an optional type without `default:`, and a required form for a
non-optional type without `default:`.

#### Scenario: defaulted form with the key absent
- **WHEN** `ConfigProperty<Int>.wireValue(from:forKey:default:)` is called for a key the reader does not hold
- **THEN** it returns the default

#### Scenario: defaulted form with the key present
- **WHEN** `ConfigProperty<Int>.wireValue(from:forKey:default:)` is called for a key the reader holds
- **THEN** it returns the reader's value, not the default

#### Scenario: optional form with the key absent
- **WHEN** `ConfigProperty<String?>.wireValue(from:forKey:)` is called for a key the reader does not hold
- **THEN** it returns `nil` without throwing

#### Scenario: required form with the key absent
- **WHEN** `ConfigProperty<String>.wireValue(from:forKey:)` is called for a key the reader does not hold
- **THEN** it throws

#### Scenario: required form with the key present
- **WHEN** `ConfigProperty<String>.wireValue(from:forKey:)` is called for a key the reader holds
- **THEN** it returns the reader's value

Pinned by: `Tests/WireConfigurationTests/ConfigPropertyTests.swift` (`defaultedFormFallsBackWhenAbsent`, `defaultedFormReadsThePresentValue`, `optionalFormYieldsNilWhenAbsent`, `optionalFormYieldsThePresentValue`, `requiredFormThrowsWhenAbsent`, `requiredFormReadsThePresentValue`).

### Requirement: Each form reads through the matching reader method
The static `wireValue(from:forKey:…)` for a type SHALL call the `ConfigReader` method of the same
family for that type: the `default:` overload for the defaulted form, the plain optional-returning
method for the optional form, and the `required` method for the required form. Only the required
form SHALL be declared `throws`.

#### Scenario: an Int site in each form
- **WHEN** the three `Int` overloads of `wireValue` are invoked
- **THEN** they call `int(forKey:isSecret:default:)`, `int(forKey:isSecret:)` and `requiredInt(forKey:isSecret:)` respectively

Pinned by: nothing yet.

### Requirement: Six value types are supported
`@ConfigProperty` SHALL accept sites of type `Int`, `String`, `Bool`, `Double`, `[String]` and
`[Int]`, each in all three forms.

#### Scenario: a supported type
- **WHEN** a parameter of type `[String]` is annotated `@ConfigProperty(forKey: "hosts", default: [])`
- **THEN** the site compiles and reads through `stringArray(forKey:isSecret:default:)`

Pinned by: nothing yet.

### Requirement: An unsupported type is rejected at the annotation only at a parameter site
At a parameter site, support is by constrained overload on `Value`, so a site of any other type
SHALL fail to compile at the annotation. At a stored-property site the peer macro SHALL NOT check
the property's type, so an unsupported type, or a `default:` of a type other than the property's,
compiles at the annotation and fails only where the WireGen-generated
`ConfigProperty<T>.wireValue(...)` call does not type-check, which is tracked as a defect in
https://github.com/swift-wire/wire-configuration/issues/14.

#### Scenario: an unsupported type at a parameter site
- **WHEN** a parameter of type `URL` is annotated `@ConfigProperty(forKey: "endpoint")`
- **THEN** compilation fails at the annotation because no `ConfigProperty` initialiser is constrained to `URL`

#### Scenario: an unsupported type at a stored-property site
- **WHEN** a stored property is declared `@ConfigProperty(forKey: "endpoint") let endpoint: URL`
- **THEN** the attribute compiles, because the peer macro overload without `default:` names no value type

Pinned by: nothing yet.

### Requirement: The `isSecret` flag is forwarded on every form
Every initialiser and every `wireValue` overload SHALL take `isSecret: Bool` defaulting to `false`,
and `wireValue` SHALL forward it to the reader call so redaction is the reader's decision.

#### Scenario: a secret String parameter
- **WHEN** a `String` parameter is written `@ConfigProperty(forKey: "db.password", isSecret: true) password: String`
- **THEN** the synthesised producer calls `ConfigProperty<String>.wireValue(from: _wireProvider, forKey: "db.password", isSecret: true)`

#### Scenario: forwarding to the reader
- **WHEN** `ConfigProperty<String>.wireValue(from:forKey:isSecret:)` is called with `isSecret: true`
- **THEN** it passes `isSecret: true` to `requiredString(forKey:isSecret:)`

Pinned by: nothing yet.

### Requirement: The annotation attaches to a parameter or a stored property
`@ConfigProperty` SHALL ship as two declarations sharing one name: a property wrapper, which is what
attaches to a function or initialiser parameter, and a peer macro, which is what attaches to a
stored property and admits `let` as well as `var`. The macro SHALL expand to nothing.

#### Scenario: a parameter site
- **WHEN** a function parameter is written `@ConfigProperty(forKey: "port", default: 8080) port: Int` and the function is called with `port: 42`
- **THEN** the parameter carries `42` and the wrapper's `wrappedValue` is `42`

#### Scenario: a stored `let` site
- **WHEN** a `@Singleton` declares `@Inject @ConfigProperty(forKey: "maxConnections", default: 10) let maxConnections: Int`
- **THEN** the declaration compiles, the macro adds no peers, and Wire reads the attribute syntactically exactly as it does at a parameter site

Pinned by: `Tests/WireConfigurationTests/ConfigPropertyTests.swift` (`attachmentCarriesTheValue`, `attachmentFormCarriesTheValue`). The stored-property site is pinned by nothing yet.

### Requirement: The wrapper carries a value and never resolves one
The property wrapper SHALL be constructed only with `wrappedValue:` plus the annotation's
arguments, SHALL expose `wrappedValue` as read-only, and SHALL NOT read configuration itself.
Reading SHALL happen only through the static `wireValue(from:…)` entry points, which construct no
wrapper.

#### Scenario: constructing the wrapper directly
- **WHEN** `ConfigProperty(wrappedValue: 42, forKey: "port", default: 8080)` is constructed
- **THEN** `wrappedValue` is `42` and no reader is consulted

Pinned by: `Tests/WireConfigurationTests/ConfigPropertyTests.swift` (`attachmentFormCarriesTheValue`).

### Requirement: Wire is told about the annotation through an injection rewrite
The package SHALL declare `wireConfigPropertyAnnotation` as a `WireAdapterAnnotationV1` for
annotation `ConfigProperty` with capability `.rewritesInjection(provider: "ConfigReader",
selector: .labelled("reader"))`. For each distinct combination of the annotation's arguments and the
site's type, WireGen SHALL synthesise one producer, shared by every site with that combination. The
producer depends on a `ConfigReader` binding, keyed by the `reader:` argument when one is written,
and returns the result of
`ConfigProperty<Value>.wireValue(from: <reader>, <the annotation's arguments verbatim, minus reader:>)`.

#### Scenario: a defaulted String site in a `@Provides` function
- **WHEN** a `@Provides` function declares `@ConfigProperty(forKey: "couchdb.host", default: "localhost") host: String`
- **THEN** the generated graph binds `String` for that site through a producer whose body is `try _wireRewritten(ConfigProperty<String>.wireValue(from: _wireProvider, forKey: "couchdb.host", default: "localhost"))`, where `_wireRewritten` is a private helper WireGen emits so the `try` is correct whether or not the selected overload throws

Pinned by: nothing yet in this repository. The rewrite pass itself is pinned in swift-wire by [InjectionRewriteTests.swift](https://github.com/swift-wire/swift-wire/blob/main/Tests/WireGenCoreTests/InjectionRewriteTests.swift) and the [InjectionRewriteHarness](https://github.com/swift-wire/swift-wire/tree/main/InjectionRewriteHarness) package, which use a stand-in annotation of the same shape rather than `@ConfigProperty`.

### Requirement: Sendability follows the value
`ConfigProperty<Value>` SHALL be `Sendable` when `Value` is `Sendable`.

#### Scenario: an Int wrapper
- **WHEN** a `ConfigProperty<Int>` is passed across an isolation boundary
- **THEN** it compiles without a sendability diagnostic

Pinned by: nothing yet.

## Related specifications

- [reader-binding-and-activation](../reader-binding-and-activation/spec.md)
- [swift-wire adapter-annotations](https://github.com/swift-wire/swift-wire/blob/main/openspec/specs/adapter-annotations/spec.md)
