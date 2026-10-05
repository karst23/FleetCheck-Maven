# FleetCheck - Maven

FleetCheck is a Java 21 application used to demonstrate a quality-oriented build process using Maven.

The project covers dependency management, automated testing, executable JAR generation, Maven Wrapper, GitHub Actions and Software Bill of Materials (SBOM) generation.

---

## Environment

The project was developed and tested with:

- Java 21
- Apache Maven 3.9.16
- Git 2.55.0
- GitHub
- GitHub Actions

Environment verification:

```text
openjdk version "21.0.12.1"
Apache Maven 3.9.16
git version 2.55.0.windows.5
```

---

# Evidence 1 - Initial Maven build failure

The project was first built without changing the original Maven configuration.

The following command was executed:

```bash
mvn clean package
```

The build failed during compilation because the Jackson dependency required by `App.java` was not declared in `pom.xml`.

Relevant errors:

```text
[ERROR] package com.fasterxml.jackson.core.type does not exist
```

```text
[ERROR] package com.fasterxml.jackson.databind does not exist
```

The compiler also reported:

```text
cannot find symbol
symbol: class ObjectMapper
```

and:

```text
cannot find symbol
symbol: class TypeReference
```

The imports in `App.java` responsible for the compilation failure were:

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
```

Therefore, the missing application dependency was Jackson Databind.

---

# Step 2 - Add the missing application dependency

The following dependency was added to the `<dependencies>` section of `pom.xml`:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.22.2</version>
</dependency>
```

The build was then executed again:

```bash
mvn clean package
```

After adding Jackson, the application compiled successfully.

The test phase then exposed a behavioural defect.

Relevant failure:

```text
FleetServiceTest.vehicleAtServiceIntervalShouldNeedService
expected: <true> but was: <false>
```

The defect was located in `FleetService.java`.

The original condition was:

```java
return kilometresSinceService > vehicle.serviceIntervalKm();
```

This condition failed when the kilometres since the last service were exactly equal to the service interval.

It was corrected to:

```java
return kilometresSinceService >= vehicle.serviceIntervalKm();
```

After correcting the boundary condition, the build succeeded:

```text
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

## Why is this a better failure than the one from Step 1?

This is a better failure because the application now compiles successfully and Maven is able to execute the automated tests.

The failure from Step 1 was caused by a missing dependency and prevented the application from compiling.

The second failure occurred later in the build process and detected an actual behavioural defect in the application logic.

This means that the build progressed from detecting a configuration problem to validating whether the software behaves correctly.

---

# Step 3 - Inspect the Maven dependency graph

The resolved dependency graph was inspected using:

```bash
mvn dependency:tree
```

Relevant output:

```text
pt.upt.softwarequality:fleetcheck:jar:1.0.0
+- com.fasterxml.jackson.core:jackson-databind:jar:2.22.2:compile
|  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.22:compile
|  \- com.fasterxml.jackson.core:jackson-core:jar:2.22.2:compile
\- org.junit.jupiter:junit-jupiter:jar:5.14.4:test
```

The direct application dependency is:

```text
jackson-databind
```

The transitive dependencies include:

```text
jackson-core
jackson-annotations
```

`jackson-databind` is a direct dependency because it is explicitly declared in `pom.xml`.

`jackson-core` and `jackson-annotations` are transitive dependencies because Maven resolves them automatically as dependencies required by Jackson Databind.

---

# Evidence 4 - Build and execute the JAR

The default Maven package was first created with:

```bash
mvn clean package
```

The generated JAR was:

```text
target/fleetcheck-1.0.0.jar
```

Trying to execute it with:

```bash
java -jar target\fleetcheck-1.0.0.jar
```

produced:

```text
no main manifest attribute, in target\fleetcheck-1.0.0.jar
```

This showed that the normal Maven JAR was not yet a self-contained executable application.

The Maven Shade Plugin was then added to `pom.xml`.

The plugin was configured to:

- execute during the `package` phase;
- include the runtime dependencies;
- define `pt.upt.fleetcheck.App` as the main class;
- generate a shaded artifact using the `all` classifier.

The new artifact was generated as:

```text
target/fleetcheck-1.0.0-all.jar
```

The shaded JAR was executed using:

```bash
java -jar target\fleetcheck-1.0.0-all.jar
```

Application output:

```text
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

## What did the Shade plugin change compared with the default JAR?

The default Maven JAR contained the FleetCheck application classes but did not contain everything required to run the application directly.

The Maven Shade Plugin created a self-contained executable JAR.

It included the required runtime dependencies and added the `Main-Class` information to the JAR manifest.

As a result, the shaded JAR can be executed directly using:

```bash
java -jar target/fleetcheck-1.0.0-all.jar
```

---

# Step 5 - Maven Wrapper

The Maven Wrapper was generated using:

```bash
mvn wrapper:wrapper
```

The wrapper was configured to use Maven 3.9.16.

The project then contained:

```text
mvnw
mvnw.cmd
.mvn/wrapper/
```

The following property was also added to `pom.xml`:

```xml
<project.build.outputTimestamp>2026-09-27T00:00:00Z</project.build.outputTimestamp>
```

The wrapper was tested on Windows with:

```bash
mvnw.cmd clean verify
```

The build completed successfully:

```text
BUILD SUCCESS
```

## Which hidden environmental assumption did the Maven Wrapper remove?

The Maven Wrapper removes the assumption that Maven is already installed and correctly configured on every developer or CI machine.

The wrapper allows the project to use the expected Maven version automatically.

This makes the build more reproducible and consistent across different development and CI environments.

---

# Step 6 - GitHub Actions

A GitHub repository was created for the Maven version:

```text
https://github.com/karst23/FleetCheck-Maven
```

A GitHub Actions workflow was created in:

```text
.github/workflows/build.yml
```

The workflow is configured to:

- checkout the repository;
- set up Java 21;
- use the Maven cache;
- execute the Maven Wrapper;
- run the build and quality gates;
- upload the generated build evidence.

The main CI build command is:

```bash
./mvnw -B clean verify
```

The workflow completed successfully in GitHub Actions.

The uploaded workflow artifact is named:

```text
fleetcheck-build
```

The workflow includes the following build evidence:

```text
target/*-all.jar
target/site/jacoco/**
target/bom.json
```

---

# Evidence 7 - Maven SBOM

The CycloneDX Maven Plugin was added to the Maven build.

The version property used was:

```xml
<cyclonedx.version>2.9.3</cyclonedx.version>
```

The CycloneDX plugin was configured to execute during the `verify` phase and generate a JSON Software Bill of Materials.

The build was executed using:

```bash
mvnw.cmd clean verify
```

Relevant output:

```text
CycloneDX: Resolving Dependencies
CycloneDX: Creating BOM version 1.6 with 3 component(s)
CycloneDX: Writing and validating BOM (JSON): target\bom.json
BUILD SUCCESS
```

The generated SBOM is:

```text
target/bom.json
```

The SBOM contains components including:

```text
jackson-databind
jackson-core
jackson-annotations
```

## Why does the SBOM contain components that were not explicitly typed in the original dependencies section?

The SBOM represents the complete dependency graph resolved by Maven, not only the dependencies explicitly declared in `pom.xml`.

`jackson-databind` is explicitly declared as a direct dependency.

However, `jackson-core` and `jackson-annotations` are required by Jackson Databind and are resolved automatically by Maven as transitive dependencies.

Because the SBOM describes all software components used by the application, these transitive dependencies are also included.

---

# Final Maven Result

The Maven version of FleetCheck successfully demonstrates:

- Java 21 compilation;
- direct dependency declaration;
- transitive dependency resolution;
- automated testing;
- detection and correction of a behavioural defect;
- executable JAR generation using the Maven Shade Plugin;
- use of the Maven Wrapper;
- automated builds using GitHub Actions;
- CycloneDX SBOM generation.

The final executable Maven artifact is:

```text
target/fleetcheck-1.0.0-all.jar
```

The generated SBOM is:

```text
target/bom.json
```

The Maven repository is:

```text
https://github.com/karst23/FleetCheck-Maven
```