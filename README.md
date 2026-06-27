# addon-commons-demo

A small, runnable demo of [**addon-commons**](https://github.com/Jerady/addon-commons) — a lightweight
Java library that turns the standard JDK `ServiceLoader` mechanism into a simple **add-on / plugin system**.

This project shows the full add-on lifecycle end to end:

1. Define an add-on **contract** (an SPI interface).
2. Provide several **implementations**.
3. **Register** them as services.
4. **Discover and use** them at runtime — both from the classpath and from external add-on jars.

The example domain is a playful *Star Wars* string converter: each add-on converts a name from one form
to another (e.g. `"Anakin Skywalker"` → `"Darth Vader"`).

---

## Requirements

| Tool            | Version                                                                 |
|-----------------|-------------------------------------------------------------------------|
| JDK             | 11–21 (the build runs on these; compiled output targets Java 8 bytecode)|
| Gradle          | Provided via the wrapper (`./gradlew`), Gradle 8.5 — no local install   |
| `addon-commons` | `16.0.0`                                                                 |

> **Dependency note:** `de.jensd:addon-commons` is **not on Maven Central or JitPack**. The build resolves it
> from your local Maven repository (`~/.m2`). If a clean build fails to resolve it, install/publish
> `addon-commons` to your local `.m2` first (clone [addon-commons](https://github.com/Jerady/addon-commons)
> and run `./gradlew publishToMavenLocal`).

---

## Quick start

```bash
./gradlew build          # compile + run all tests
./gradlew test           # run tests only
./gradlew clean          # remove build/
./gradlew publishToMavenLocal   # publish this artifact (+ sources & javadoc jars) to ~/.m2
```

Run a single test method:

```bash
./gradlew test --tests "de.jensd.addon.AddonRegistryTest.testConverter"
```

---

## How the add-on mechanism works

There are three moving parts. Understanding how they fit together is the whole point of this demo.

### 1. The contract (SPI interface)

`StringConverter` is the service interface. It extends the library's `AddOn` marker interface, which is
what makes it discoverable by the registry.

```java
package de.jensd.addon.demo.converter;

import de.jensd.addon.AddOn;

public interface StringConverter extends AddOn {
    String convert(String value);
    String name();   // unique key used to look an add-on up
}
```

### 2. The implementations

Three converters implement the contract. Each returns a unique `name()` used as its lookup key:

| Implementation        | `name()`              | Behaviour (example)                          |
|-----------------------|-----------------------|----------------------------------------------|
| `DarkSideConverter`   | `DarkSideConverter`   | `"Anakin Skywalker"` → `"Darth Vader"`       |
| `LightSideConverter`  | `LightSideConverter`  | `"Darth Vader"` → `"Anakin Skywalker"`       |
| `CastConverter`       | `CastConverter`       | `"Luke"` → `"Mark Hamill"`                    |

### 3. Service registration

This is the step that is easy to forget. Implementations are registered with the JDK `ServiceLoader` via a
provider-configuration file named after the **fully-qualified interface name**:

```
src/main/resources/META-INF/services/de.jensd.addon.demo.converter.StringConverter
```

```text
de.jensd.addon.demo.converter.DarkSideConverter
de.jensd.addon.demo.converter.LightSideConverter
de.jensd.addon.demo.converter.CastConverter
```

> ⚠️ **An implementation that is not listed in this file will not be discovered.** Adding a new converter
> always requires adding its fully-qualified class name here.

### 4. Discovery at runtime

The library's `AddOnRegistryServiceLoader` finds and instantiates the registered add-ons. There are two
discovery modes:

**a) From the classpath** — picks up everything registered in `META-INF/services`:

```java
AddOnRegistryServiceLoader registry = new AddOnRegistryServiceLoader();
List<StringConverter> converters = registry.getAddOns(StringConverter.class);

Map<String, StringConverter> byName = converters.stream()
        .collect(Collectors.toMap(StringConverter::name, c -> c));

byName.get("DarkSideConverter").convert("Anakin Skywalker"); // -> "Darth Vader"
```

**b) From external add-on jars** — scans a directory for add-on jars at runtime. The directory is configured
through a system property (its name is exposed as
`AddOnRegistryServiceLoader.ADDON_LOOKUP_PATH_PROPERTY_NAME`):

```java
System.setProperty(AddOnRegistryServiceLoader.ADDON_LOOKUP_PATH_PROPERTY_NAME, "path/to/addons");
AddOnRegistryServiceLoader registry = new AddOnRegistryServiceLoader();
List<URL> addOnUrls = registry.lookupAddOnUrls();
```

`src/test/resources/addon-commons-demo-0.0.5.addon.jar` is a pre-built add-on jar used to exercise this mode
in the tests (the project itself, packaged as an add-on).

---

## Adding your own converter

1. Create a class implementing `StringConverter` under
   `src/main/java/de/jensd/addon/demo/converter/`.
2. Implement `convert(String)` and return a unique value from `name()`.
3. Append its fully-qualified class name to
   `src/main/resources/META-INF/services/de.jensd.addon.demo.converter.StringConverter`.
4. Run `./gradlew test` — `AddonRegistryTest.testLoadExtensions` asserts the number of discovered
   converters, so update that assertion if you change the count.

---

## Project layout

```
src/
├── main/
│   ├── java/de/jensd/addon/demo/converter/
│   │   ├── StringConverter.java        # the SPI contract (extends AddOn)
│   │   ├── DarkSideConverter.java
│   │   ├── LightSideConverter.java
│   │   └── CastConverter.java
│   └── resources/
│       ├── META-INF/services/
│       │   └── de.jensd.addon.demo.converter.StringConverter   # service registration
│       └── log4j2.xml                  # console logging config
└── test/
    ├── java/de/jensd/addon/
    │   └── AddonRegistryTest.java       # end-to-end usage; best entry point
    └── resources/
        └── addon-commons-demo-0.0.5.addon.jar   # sample add-on jar for jar-based discovery
```

`AddonRegistryTest` is the best place to start reading — it demonstrates both discovery modes and the
converter behaviour in one file.

---

## Conventions

- The project `version` is `0.0.5.addon`. The `.addon` suffix is the add-on jar extension convention the
  library uses when discovering jar-based add-ons.
- Every source file carries the **Apache 2.0** license header — keep it on new files.

---

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
Copyright © Jens Deters — http://www.jensd.de
