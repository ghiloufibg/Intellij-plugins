# Spring Boot + Hibernate/JPA + @Scheduled → GraalVM Native Image
## The Complete Enterprise Guide

A field guide combining official documentation, real-world enterprise fixes from the Hibernate/Spring communities, and hard-won troubleshooting knowledge. Target audience: teams running Spring Boot 3.x/4.x with Spring Data JPA, Hibernate ORM, and scheduled tasks who need a reliable, repeatable native compilation pipeline.

---

## 1. Why This Is Hard (Read This First)

GraalVM Native Image works on a **closed-world assumption**: everything the application will ever do must be known at build time. Three things your stack does violate that assumption by default:

| Component | What it does at runtime | Why native breaks |
|---|---|---|
| Hibernate | Generates entity proxies and optimized accessors via **ByteBuddy** (`ClassLoader.defineClass()`) | Native images cannot define new classes at runtime — ever |
| JPA / Spring Data | Reflection on entities, repositories, DTO constructors | Reflection targets must be registered at build time |
| `@Scheduled` | Mostly fine — but programmatic/reflective scheduling and SpEL-heavy cron config can break | Dynamic method resolution isn't visible to static analysis |

The enterprise-standard answer is **not** to hand-write `reflect-config.json` for everything. It's to shift all dynamic behavior to **build time**:

1. **Spring AOT** generates bean definitions, proxies, and reflection hints during the build.
2. **Hibernate build-time enhancement** replaces runtime ByteBuddy proxy generation.
3. **Reachability metadata repository** supplies community-maintained config for libraries.
4. **RuntimeHints / tracing agent** fill only the residual gaps.

---

## 2. Version Matrix — Get This Right Before Anything Else

Version misalignment is the single most common root cause of native runtime failures in enterprise setups.

**Rules:**

- **Spring Boot 3.2.4+ minimum**, ideally the latest 3.3/3.4/3.5 patch (or Boot 4.x). Native support matured significantly after 3.2.
- **Do NOT manually override the Hibernate version** that Spring Boot's BOM manages. There was a documented regression window: Hibernate 6.4.3+/6.2.21+ broke Spring Framework's native substitution for the `BytecodeProvider`; it was fixed on Spring's side in Framework 6.1.5/6.0.18 (Boot 3.2.4/3.1.10). If your `pom.xml` pins `hibernate.version`, that pin is suspect #1.
- **GraalVM version ≥ your Java version.** Use a GraalVM JDK matching your `<java.version>` (e.g., GraalVM for JDK 21 with Java 21). Oracle GraalVM or GraalVM CE both work; Liberica NIK is a common enterprise alternative used by Buildpacks.
- **ByteBuddy version ≥ your class-file version.** "Unsupported class file major version 65/69" at build time = ByteBuddy older than your JDK. Managed automatically if Boot and JDK versions are aligned.

**Verification commands:**

```bash
mvn dependency:tree -Dincludes=org.hibernate.orm:hibernate-core
mvn dependency:tree -Dincludes=net.bytebuddy:byte-buddy
mvn help:evaluate -Dexpression=hibernate.version -q -DforceStdout
java -version   # must say GraalVM
```

---

## 3. The Canonical Build Setup (Maven)

### 3.1 POM structure

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.x</version> <!-- latest patch -->
</parent>

<properties>
    <java.version>21</java.version>
    <!-- Do NOT set hibernate.version unless you have a documented reason -->
</properties>

<build>
    <plugins>
        <!-- 1. Spring Boot plugin: triggers process-aot automatically in native profile -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- 2. GraalVM native build tools -->
        <plugin>
            <groupId>org.graalvm.buildtools</groupId>
            <artifactId>native-maven-plugin</artifactId>
        </plugin>

        <!-- 3. Hibernate build-time enhancement — THE key piece for JPA on native -->
        <plugin>
            <groupId>org.hibernate.orm.tooling</groupId>
            <artifactId>hibernate-enhance-maven-plugin</artifactId>
            <version>${hibernate.version}</version>
            <executions>
                <execution>
                    <id>enhance</id>
                    <goals><goal>enhance</goal></goals>
                    <configuration>
                        <enableLazyInitialization>true</enableLazyInitialization>
                        <enableDirtyTracking>true</enableDirtyTracking>
                        <enableAssociationManagement>true</enableAssociationManagement>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**Multi-module note (relevant for hexagonal architectures):** the `hibernate-enhance-maven-plugin` must run in the module that **contains the `@Entity` classes** (typically your domain or persistence adapter module), not in the application/boot module. Verify with build log lines: `Enhancing [com.yourco.domain.MyEntity]`. If entities live in a module without the plugin, they silently ship un-enhanced and fail at native runtime.

### 3.2 Build commands

```bash
# Local native binary (requires local GraalVM):
mvn -Pnative native:compile

# Container image via Buildpacks (no local GraalVM needed — good for
# corporate machines where installing GraalVM is restricted):
mvn -Pnative spring-boot:build-image

# Run tests INSIDE the native image:
mvn -PnativeTest test
```

The `native` profile is defined by `spring-boot-starter-parent` — it wires `process-aot` before `native:compile`. **This ordering is what makes everything work.** Custom pipelines that call `native-image` directly and skip `process-aot` are the #1 enterprise failure mode (see §6).

### 3.3 Application properties for native

```properties
# Never use ddl-auto: update/create with native — Hibernate tries to open a
# JDBC connection during AOT processing and the build fails.
# Use Flyway/Liquibase for schema management (which is enterprise standard anyway).
spring.jpa.hibernate.ddl-auto=validate

# Optional but recommended once build-time enhancement is in place:
spring.jpa.properties.hibernate.bytecode.provider=none
```

---

## 4. The Hibernate / ByteBuddy Problem in Depth

### 4.1 Symptom catalogue

| Error at native runtime | Meaning |
|---|---|
| `BytecodeProvider: ...bytebuddy.BytecodeProviderImpl — Unable to get public no-arg constructor` | Hibernate's ServiceLoader loaded ByteBuddy; it can't initialize in a native image |
| `UnsupportedFeatureError: No classes have been predefined during the image build to load from bytecodes at runtime` | ByteBuddy attempted `defineClass()` at runtime |
| `UnsupportedOperationException: Defining new classes at runtime is not supported` | Same root cause, different call path (MethodHandles.Lookup.defineClass) |
| `ClassNotFoundException` / `NoSuchMethodException` on `MyEntity$HibernateProxy$...` | Runtime lazy-loading proxy needed but never generated — entity not enhanced at build time |

### 4.2 Root cause

Hibernate's `AggregatedServiceLoader` discovers the ByteBuddy `BytecodeProviderImpl` via classpath scanning of `META-INF/services/`, **regardless of the `hibernate.bytecode.provider` property** — the ServiceLoader runs before the property is evaluated. This behavior exists across all Hibernate 6.x and 7.x versions.

### 4.3 How Spring fixes it (when your build is standard)

Spring Framework ships two mechanisms as part of AOT/native support:

1. A **GraalVM substitution** that forces the bytecode provider to `none` when running in a native image (`NativeDetector.inNativeImage()`).
2. A **native-image property** baked into Spring's metadata: `-H:ServiceLoaderFeatureExcludeServices=org.hibernate.bytecode.spi.BytecodeProvider` — this removes the ByteBuddy provider from ServiceLoader discovery entirely.

**Both only apply if Spring's AOT processing actually runs and Spring's native-image.properties are on the image classpath.** Quarkus does the equivalent internally, which is why Hibernate maintainers point to framework-level substitution as the supported path.

### 4.4 Fix ladder (apply in order, stop when it works)

**Level 1 — Use the standard pipeline (fixes ~90% of enterprise cases)**
- Build with `mvn -Pnative native:compile` or `spring-boot:build-image`.
- Confirm `spring-boot:process-aot` appears in the build log.
- Confirm generated sources appear under `target/spring-aot/`.
- Align versions per §2.

**Level 2 — Build-time enhancement + provider none**
- Ensure `hibernate-enhance-maven-plugin` runs on the entity module (check for `Enhancing [...]` log lines).
- Set `spring.jpa.properties.hibernate.bytecode.provider=none`.
- These are complementary: enhancement replaces what ByteBuddy would have generated (lazy proxies, dirty tracking); `none` tells Hibernate not to try.

**Level 3 — Explicit ServiceLoader exclusion**

If a custom build prevents Spring's own flag from applying, add it yourself:

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <buildArgs>
            <buildArg>-H:ServiceLoaderFeatureExcludeServices=org.hibernate.bytecode.spi.BytecodeProvider</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```

**Level 4 — Documented "nuclear" workarounds (constrained builds only: AWS Lambda custom runtimes, locked-down Docker pipelines, Gradle setups bypassing AOT)**

Real teams have shipped production systems (13+ microservices in one documented case) with this combination:

1. Strip the ServiceLoader registration from the jar before native compilation:
   ```bash
   find ~/.m2/repository -name "hibernate-core-*.jar" -exec \
       zip -d {} "META-INF/services/org.hibernate.bytecode.spi.BytecodeProvider" \;
   ```
   (In Docker: run between dependency resolution and `native:compile`.)
2. Provide an override at `src/main/resources/META-INF/services/org.hibernate.bytecode.spi.BytecodeProvider`:
   ```
   org.hibernate.bytecode.internal.none.BytecodeProviderImpl
   ```
3. Belt-and-suspenders in `main()`:
   ```java
   public static void main(String[] args) {
       System.setProperty("hibernate.bytecode.provider", "none");
       SpringApplication.run(Application.class, args);
   }
   ```
4. Register the `none` provider for reflection:
   ```json
   { "name": "org.hibernate.bytecode.internal.none.BytecodeProviderImpl",
     "allDeclaredConstructors": true, "allDeclaredMethods": true }
   ```

An alternative middle ground used successfully on AWS Lambda: build an **uber-jar with maven-shade-plugin**, use a `ServicesResourceTransformer` + filter to drop the ByteBuddy service entry, and compile the native image from the uber-jar instead of the classpath.

> ⚠️ Hibernate maintainers consider Level 4 a workaround for broken builds, not a recommended pattern. If you reach Level 4, also file a ticket to fix your pipeline so Levels 1–3 work. Track HHH-19530 (ByteBuddy moving to a separate artifact) — this whole class of problem should eventually disappear.

---

## 5. JPA / Entity Reflection: Closing the Remaining Gaps

Spring AOT registers entities, repositories, and most Jackson bindings automatically. What it typically **misses**:

**Constructor-expression DTOs** (`SELECT new com.x.OrderSummary(...)`) — register them:

```java
@Configuration
@RegisterReflectionForBinding({OrderSummary.class, ReportRow.class})
class NativeReflectionConfig {}
```

**Custom types, AttributeConverters, UserTypes, event listeners loaded by name, JDBC drivers referenced via `Class.forName`** — use RuntimeHints:

```java
@Configuration
@ImportRuntimeHints(PersistenceHints.class)
class PersistenceConfig {}

class PersistenceHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader cl) {
        hints.reflection().registerType(MoneyConverter.class, MemberCategory.values());
        hints.resources().registerPattern("db/migration/*.sql"); // Flyway scripts
        hints.serialization().registerType(CacheKey.class);      // if Java serialization used
    }
}
```

**Named native queries returning scalar arrays, criteria queries with `Tuple`** — usually fine; entity-mapped results require the entity to be registered (automatic).

**Reachability metadata repository:** enabled by default in native-build-tools. It supplies community-maintained config for Hibernate, HikariCP, PostgreSQL/MySQL drivers, Jackson, etc. Keep the plugin version current — metadata coverage improves constantly.

**Tracing agent — discovery tool, not a permanent fixture:**

```bash
java -agentlib:native-image-agent=config-output-dir=target/native-agent-config \
     -jar target/app.jar
# Exercise every path: REST endpoints, ALL scheduled jobs (trigger manually or
# temporarily set aggressive cron), error paths, health checks. Then Ctrl+C.
```

Enterprise practice: run the agent **once** to discover gaps, translate findings into `RuntimeHints` code (reviewable, refactor-safe, typed), and do **not** commit raw agent JSON — it goes stale silently and bloats the image.

---

## 6. @Scheduled on Native

`@Scheduled` with fixed rates/delays and cron expressions (including `${...}` from config) works out of the box — Spring AOT processes the annotations at build time. The gotchas:

- **Programmatic scheduling via reflection** (looking up methods by name, dynamic `Runnable` class loading) breaks. Refactor to lambda/method references on Spring beans.
- **SpEL in schedules** beyond simple property placeholders can require SpEL AOT support; keep expressions simple.
- **`@Async` + `@Scheduled` combos** — proxies are generated at build time by AOT; ensure the methods are on Spring beans, not called internally (self-invocation bypasses proxies on JVM too, but failures surface harder on native).
- **ShedLock / Quartz users:** Quartz uses reflection heavily; check the reachability metadata repo and Quartz's native docs, or prefer plain `@Scheduled` + ShedLock (JDBC lock provider works well on native) for clustered deployments.
- **Verify jobs actually fire** in your native smoke test — a missing hint often fails silently by never invoking the job rather than crashing.

---

## 7. Known Environmental Gotchas (from Spring's Official Known-Issues List)

- **JAXB/freetype:** Hibernate depends on `jaxb-runtime`; a class inside JAXB touches `javax.imageio`, which initializes the graphics subsystem. On Linux build machines/containers you may need `libfreetype` installed — a bizarre-looking build failure with an easy fix.
- **Gradle + Kotlin DSL + Hibernate enhancement:** ClassCastException when configuring bytecode enhancement (tracked as HHH-15707). Maven or Groovy DSL avoid it.
- **`ddl-auto=update`:** fails during `process-aot` because Hibernate wants a JDBC connection at AOT time. Use `validate` + Flyway/Liquibase (see §3.3).
- **Logback/Log4j2:** Logback works; complex custom appenders may need hints. Log4j2 async logging historically needed extra config.
- **Corporate proxies/certificates:** Buildpacks downloads a builder image and GraalVM internally — configure Docker's proxy and trusted CAs, or use a mirrored builder image in your internal registry. For direct `native:compile`, GraalVM download must go through your artifact proxy (Nexus/Artifactory raw proxy of the GraalVM releases works well).

---

## 8. Testing & CI/CD — The Enterprise Pipeline Shape

Native compilation is expensive (minutes, 4+ GB RAM, multiple CPUs). The standard enterprise pattern splits pipelines:

```
PR pipeline (fast, every commit):
  ├─ mvn verify                      # unit + integration tests on JVM
  ├─ mvn spring-boot:process-aot     # cheap early warning: AOT processing
  │                                  # failures surface here, not at release
  └─ Testcontainers integration tests against real DB

Nightly / release pipeline (slow, gated):
  ├─ mvn -Pnative spring-boot:build-image   # containerized native build
  ├─ mvn -PnativeTest test                  # test suite inside native image
  └─ Smoke tests against the native container:
       ├─ startup < N ms assertion
       ├─ health endpoint
       ├─ one CRUD round-trip per aggregate (exercises entity reflection)
       └─ force-trigger every scheduled job and assert side effects
```

Additional practices:

- **Runner sizing:** dedicate builders with ≥ 8 GB RAM; cap with `-J-Xmx7g` in buildArgs if the runner OOMs.
- **Cache the Maven repo AND the Buildpacks builder image** in CI to keep native builds under ~10 minutes.
- **Ship both artifacts** during migration: JVM image as fallback, native as primary. Rollback = redeploy JVM image.
- **Consider CDS as a middle ground:** Spring Boot 3.3+ supports Class Data Sharing (`BP_JVM_CDS_ENABLED=true` with Buildpacks) — ~1.5–3× faster JVM startup with zero hint maintenance. Some enterprises adopt CDS for services where the native investment isn't justified and keep native for scale-to-zero/serverless workloads.
- **Know the trade-offs:** representative published benchmarks for a JPA + PostgreSQL REST service show native winning dramatically on startup (tens of ms vs seconds) and memory (~2–4× less), while JVM retains higher peak throughput after warm-up. Long-running batch/high-throughput services may be better on JVM; short-lived, autoscaled, or serverless workloads strongly favor native.

---

## 9. Troubleshooting Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `BytecodeProviderImpl — no-arg constructor` at startup | ByteBuddy loaded via ServiceLoader | §4.4 ladder; check AOT ran |
| `No classes have been predefined...` | Runtime class definition attempted | Build-time enhancement + provider `none` |
| `ClassNotFoundException: X$HibernateProxy$...` | Entity not enhanced | Enhancement plugin on the right module |
| Works on JVM, entity fields null/stale on native | Dirty tracking not enhanced | `enableDirtyTracking` + rebuild |
| `Unsupported class file major version NN` at build | ByteBuddy < JDK version | Align Boot version; force newer byte-buddy |
| AOT phase fails wanting a JDBC connection | `ddl-auto=update` | `validate` + Flyway/Liquibase |
| DTO projection fails: no suitable constructor | Constructor expression DTO unregistered | `@RegisterReflectionForBinding` |
| Scheduled job never runs, no error | Reflective scheduling / missing hint | Plain `@Scheduled` on a bean; check AOT output |
| `MissingReflectionRegistrationError` naming a class | Any reflective access AOT missed | Targeted RuntimeHint; agent to discover |
| Random `Resource not found` (SQL files, templates) | Resources not registered | `hints.resources().registerPattern(...)` |
| Build fails on `awt`/`freetype` | JAXB → imageio chain | Install libfreetype in build image |
| Native build OOMs in CI | Under-provisioned runner | ≥8 GB RAM runner; `-J-Xmx` cap |

**Debugging tactic:** every `MissingReflectionRegistrationError` names the exact class/member. Fix them one at a time with targeted hints — resist the urge to register whole packages with `allDeclared*` (image size and build time balloon).

---

## 10. Migration Checklist (Copy Into Your Ticket)

- [ ] Spring Boot on latest patch of 3.3+/4.x; no manual `hibernate.version` pin
- [ ] GraalVM JDK matching `java.version` (or Buildpacks strategy chosen)
- [ ] `native-maven-plugin` added; builds via `-Pnative` profile only
- [ ] `hibernate-enhance-maven-plugin` on the entity module(s); `Enhancing [...]` visible in logs
- [ ] `ddl-auto=validate` + Flyway/Liquibase
- [ ] `hibernate.bytecode.provider=none` set
- [ ] Constructor-expression DTOs registered via `@RegisterReflectionForBinding`
- [ ] Custom converters/types covered by `RuntimeHintsRegistrar`
- [ ] Tracing agent run once; findings converted to hints (no raw JSON committed)
- [ ] All scheduled jobs verified firing in native smoke test
- [ ] CI split: JVM tests per-PR (+ `process-aot`), native build + `nativeTest` nightly/release
- [ ] JVM fallback image retained during rollout
- [ ] Proxy/CA config for Buildpacks in corporate network documented

---

## 11. Sources & Further Reading

**Official documentation**
- Spring Boot GraalVM Native Images reference — docs.spring.io/spring-boot (native-image chapter)
- Spring Boot with GraalVM known issues wiki — github.com/spring-projects/spring-boot/wiki/Spring-Boot-with-GraalVM
- GraalVM Reachability Metadata — github.com/oracle/graalvm-reachability-metadata
- Hibernate ORM bytecode enhancement guide — hibernate.org/orm/documentation
- GraalVM Native Build Tools — graalvm.github.io/native-build-tools

**Key issue trackers (the real-world history of this problem)**
- spring-framework#32311 — Hibernate native support broken by HHH-17643; Spring's ServiceLoader-exclusion fix
- spring-boot#35659 — Native image fails with Hibernate on Boot 3.1; version-alignment regression
- HHH-19530 — Moving ByteBuddy to a separate artifact (the long-term fix)
- HHH-15707 — Gradle Kotlin DSL enhancement ClassCastException
- raphw/byte-buddy#1560 — "Defining new classes at runtime is not supported" analysis

**Community solutions (Hibernate Discourse)**
- "Hibernate ByteBuddy + GraalVM Native Image with Spring Boot (Full Workaround)" — the jar-patch approach, 13 services in production
- "How to correctly disable Byte Buddy for GraalVM Native Image" — uber-jar/service-override approach for AWS Lambda
- Vadym Kazulkin's AWS Lambda + Spring Boot 4 + JPA native examples — github.com/Vadym79/aws-lambda-java-25-spring-boot-4

---

*Last updated: August 2026. The native ecosystem moves fast — re-verify version-specific claims against release notes when Spring Boot or Hibernate major/minor versions change.*
