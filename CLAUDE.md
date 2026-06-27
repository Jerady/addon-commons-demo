# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo / reference implementation of [addon-commons](https://github.com/Jerady/addon-commons), a small Java library for a `ServiceLoader`-based plugin (add-on) mechanism. This repo shows how to *consume* that library: define an add-on contract, provide implementations, register them as services, and discover them at runtime.

## Build & test

Uses the Gradle wrapper (Gradle 8.5, runs on Java 11–21; build targets `sourceCompatibility 1.8`).

```bash
./gradlew build          # compile + run tests
./gradlew test           # run tests only
./gradlew clean          # wipe build/
./gradlew publishToMavenLocal   # publish artifact (incl. sources & javadoc jars) to ~/.m2
```

Run a single test method:

```bash
./gradlew test --tests "de.jensd.addon.AddonRegistryTest.testConverter"
```

Dependencies come from Maven Central / mavenLocal; the core one is `de.jensd:addon-commons` (currently 16.0.0). `AddOn`, `AddOnRegistryServiceLoader`, and the lookup-path property live in that library, **not** in this repo.

## Architecture

The add-on pattern here has three moving parts:

1. **Contract** — `StringConverter` (`src/main/java/.../converter/StringConverter.java`) extends the library's `AddOn` marker interface. This is the SPI type that gets discovered.

2. **Implementations** — `DarkSideConverter`, `LightSideConverter`, `CastConverter` implement `StringConverter`. Each returns a `name()` used as its lookup key.

3. **Service registration** — `src/main/resources/META-INF/services/de.jensd.addon.demo.converter.StringConverter` is the standard Java `ServiceLoader` file listing the fully-qualified implementation classes. **Adding a new converter requires adding its FQN to this file**, or it won't be discovered.

Discovery at runtime goes through the library's `AddOnRegistryServiceLoader`:
- `getAddOns(StringConverter.class)` returns all registered implementations (classpath `ServiceLoader`).
- `lookupAddOnUrls()` scans a directory for add-on jars; the directory is set via the system property named by `AddOnRegistryServiceLoader.ADDON_LOOKUP_PATH_PROPERTY_NAME`. `AddonRegistryTest.testLoadExtensionsFromJar` points it at `build/resources/test`, where `src/test/resources/addon-commons-demo-0.0.5.addon.jar` is copied — this is how the library loads add-ons packaged as external jars.

`AddonRegistryTest` is the best entry point for understanding the whole flow end to end.

## Conventions

- Project `version` is `0.0.5.addon` — the `.addon` suffix is the add-on jar extension convention the library uses for jar-based discovery.
- All source files carry the Apache 2.0 license header (see existing files for the exact block); keep it on new files.
