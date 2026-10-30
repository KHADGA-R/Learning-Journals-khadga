# LearnHub Java & Spring Boot Backend — Lesson 1

<text color="secondary" size="sm">Java Development Environment, JDK 21, IntelliJ IDEA, Maven, JVM/JDK/JRE, and Your First Java Program</text>

<row align="center" gap={2} wrap="wrap">
  <badge color="secondary">Lesson 1 of 24</badge>
  <badge color="secondary">Experienced Java Track</badge>
  <badge color="secondary">Steady Pace</badge>
</row>

Welcome to the backend development phase of LearnHub. In the previous Angular course, we built the frontend for a course marketplace. Now we'll create the Java foundation for a real backend using Spring Boot.

Because you're already experienced with Java, we'll move relatively quickly through basic syntax and focus on **environment correctness, Java 21 features, Maven, project structure, and backend engineering practices**.

<box border radius="lg" padding={3} gap={2}>
  **Lesson outcomes**

  By the end, you should be able to:

  - Install and verify JDK 21, IntelliJ IDEA, and Maven on Windows.
  - Explain JVM, JDK, JRE, compilation, and execution.
  - Configure `JAVA_HOME` and diagnose multiple-JDK issues.
  - Build a Maven-based Java project from the command line.
  - Write a small, documented LearnHub course catalog program.
  - Compile, package, test, and run your application.
  - Prepare your workstation for Spring Boot in Lesson 5.
</box>

<text color="secondary" size="sm">Estimated time: 90–120 minutes, plus optional exercises. Suggested schedule: complete three lessons per week, with additional time for implementation.</text>

## 1. Understand the tools before installing them

<box border radius="lg" padding={3} gap={3}>
  <box gap={1}>
    **How Java code becomes a running application**
    <text color="secondary" size="xs">Simplified compilation and execution pipeline</text>
  </box>
  <box background="surface-secondary" radius="md" padding={3} align="center" gap={1}>
    <icon name="file-code" color="secondary"/>
    **Java source**
    <text color="secondary" size="xs">`Main.java`</text>
  </box>
  <row justify="center" gap={2} align="center">
    <icon name="arrow-down" color="secondary"/>
    <text color="secondary" size="xs">`javac` compiler</text>
  </row>
  <box border radius="md" padding={3} align="center" gap={1}>
    <icon name="file-code-2" color="secondary"/>
    **Java bytecode**
    <text color="secondary" size="xs">`Main.class`</text>
  </box>
  <row justify="center" gap={2} align="center">
    <icon name="arrow-down" color="secondary"/>
    <text color="secondary" size="xs">Class loading, verification, interpretation / JIT</text>
  </row>
  <box background="surface-secondary" radius="md" padding={3} align="center" gap={1}>
    <icon name="cpu" color="secondary"/>
    **Java Virtual Machine (JVM)**
    <text color="secondary" size="xs">Executes bytecode and manages memory</text>
  </box>
  <row justify="center"><icon name="arrow-down" color="secondary"/></row>
  <box border radius="md" padding={3} align="center" gap={1}>
    <icon name="monitor" color="secondary"/>
    **Running LearnHub program**
  </box>
</box>

| Term | Meaning | Main responsibility |
|---|---|---|
| JVM | Java Virtual Machine | Executes Java bytecode, handles garbage collection and runtime services |
| JRE | Java Runtime Environment | JVM and supporting runtime libraries needed to run Java programs |
| JDK | Java Development Kit | Runtime plus compiler, debugger, packaging and development tools |
| `javac` | Java compiler | Compiles `.java` files into `.class` bytecode |
| `java` | Java launcher | Starts the JVM and runs Java applications |
| Maven | Build and dependency tool | Resolves libraries, compiles, tests and packages projects |

A modern JDK already contains the runtime components needed to execute Java programs. For this course, install a **JDK**, not a standalone JRE.

### Why Java 21?

Java 21 is a long-term support release and supports modern features such as records, switch expressions, pattern matching, and virtual threads. We'll use records for response DTOs later and introduce concurrency features when they're relevant.

## 2. Prepare your Windows workstation

Open **PowerShell** or **Windows Terminal**.

Check the tools you may already have:

```powershell
java -version
javac -version
mvn -version
git --version
winget --version
```

Some commands may report that they are not recognized. That's normal before installation.

Also inspect the executable paths:

```powershell
Get-Command java -ErrorAction SilentlyContinue
Get-Command javac -ErrorAction SilentlyContinue
Get-Command mvn -ErrorAction SilentlyContinue

where.exe java
where.exe javac
```

`where.exe` may return multiple locations if you have multiple JDKs installed. That's useful information for diagnosing version conflicts.

## 3. Install JDK 21

We'll use Eclipse Temurin JDK 21.

<row align="center" gap={3}>
  <AsyncImage query="Eclipse Adoptium Temurin Java logo" aspectRatio="1:1" maxWidth="88px"/>
  <box flex="1" gap={1}>
    **Eclipse Temurin 21**
    <text color="secondary" size="sm">OpenJDK distribution suitable for local Java development.</text>
    <text size="sm"><Link url="https://adoptium.net/" title="Official Adoptium website"/></text>
  </box>
</row>

### Step 3.1 — Install using Windows Package Manager

Run:

```powershell
winget install --id EclipseAdoptium.Temurin.21.JDK --exact
```

Follow any installation prompts. Then close and reopen PowerShell so your shell picks up updated environment variables.

### Step 3.2 — Verify the installation

```powershell
java -version
javac -version
```

Expected version family:

```text
openjdk version "21.x.x"
javac 21.x.x
```

The exact patch number may differ.

### Step 3.3 — Locate your JDK

```powershell
Get-ChildItem "C:\Program Files\Eclipse Adoptium" -Directory
```

You should find a JDK 21 installation folder. Its exact name depends on the installed release.

## 4. Configure JAVA_HOME and PATH

`JAVA_HOME` should point to the **JDK root directory**, not its `bin` folder.

Example layout:

```text
C:\Program Files\Eclipse Adoptium\
└── jdk-21.x.x-hotspot\
    ├── bin\
    │   ├── java.exe
    │   ├── javac.exe
    │   └── jar.exe
    ├── conf\
    ├── legal\
    └── lib\
```

### Step 4.1 — Set JAVA_HOME for the current session

The following discovers a JDK 21 installation under the standard Temurin directory.

```powershell
$jdk = Get-ChildItem "C:\Program Files\Eclipse Adoptium" -Directory |
    Where-Object { $_.Name -match '^jdk-21([.-]|$)' } |
    Select-Object -First 1

if (-not $jdk) {
    throw "JDK 21 not found in the expected Adoptium directory."
}

$env:JAVA_HOME = $jdk.FullName
$env:Path = "$env:JAVA_HOME\bin;$env:Path"

Write-Host "JAVA_HOME = $env:JAVA_HOME"

java -version
javac -version
```

This changes the current PowerShell session only.

### Step 4.2 — Persist JAVA_HOME for your Windows user

```powershell
[Environment]::SetEnvironmentVariable(
    "JAVA_HOME",
    $jdk.FullName,
    "User"
)
```

For PATH, use the Windows graphical editor to avoid accidentally overwriting existing entries:

1. Open **Edit environment variables for your account** from Windows Search.
2. Under User variables, confirm `JAVA_HOME` points to your JDK 21 directory.
3. Edit `Path` and add `%JAVA_HOME%\bin` if it isn't already available through an existing Java path entry.
4. Move conflicting older Java paths below the intended JDK path where necessary.
5. Open a new terminal and verify again.

Run:

```powershell
$env:JAVA_HOME
java -version
javac -version
where.exe java
```

**Important:** If `java -version` still reports Java 17 or another version, check the order of PATH entries. Don't uninstall an older JDK automatically; another project may need it.

## 5. Install IntelliJ IDEA

<row align="center" gap={3}>
  <AsyncImage query="IntelliJ IDEA JetBrains IDE logo" aspectRatio="1:1" maxWidth="88px"/>
  <box flex="1" gap={1}>
    **IntelliJ IDEA**
    <text color="secondary" size="sm">Java IDE with Maven support, debugging, code navigation and refactoring.</text>
    <text size="sm"><Link url="https://www.jetbrains.com/idea/download/" title="Official IntelliJ IDEA download"/></text>
  </box>
</row>

Install IntelliJ IDEA using the official installer, or use `winget` after confirming the available package ID on your machine:

```powershell
winget search "IntelliJ IDEA"
```

The current free IntelliJ IDEA experience is sufficient for our initial Java and Maven lessons. Some advanced Spring-specific IDE conveniences may depend on the edition and licensing, but we'll always provide command-line alternatives.

### Configure the IDE

Open IntelliJ IDEA and configure:

- Project SDK: **JDK 21**
- Maven runner JDK: **JDK 21**
- File encoding: **UTF-8**
- Git executable: your installed Git
- Auto-import: Maven dependencies, where available

Later, when you open a Spring Boot project, ensure IntelliJ uses the project SDK rather than an unrelated bundled runtime.

## 6. Install Maven

<row align="center" gap={3}>
  <AsyncImage query="Apache Maven build tool official logo" aspectRatio="5:2" maxWidth="130px" fit="contain"/>
  <box flex="1" gap={1}>
    **Apache Maven**
    <text color="secondary" size="sm">Build automation and dependency management for Java applications.</text>
    <text size="sm"><Link url="https://maven.apache.org/download.cgi" title="Official Maven download"/></text>
  </box>
</row>

You can use IntelliJ's bundled Maven support, but we'll install a command-line Maven distribution for consistent PowerShell workflows.

### Step 6.1 — Download and extract Maven

Download the current Maven 3.9.x binary ZIP from the official website. Use the binary distribution, not the source archive.

Extract it into:

```text
C:\Tools\apache-maven-3.9.x\
```

You may create the parent directory with:

```powershell
New-Item -ItemType Directory -Force C:\Tools
```

If Windows denies permission, use a folder under your home directory instead:

```powershell
New-Item -ItemType Directory -Force "$HOME\tools"
```

### Step 6.2 — Configure MAVEN_HOME

For an installation under `C:\Tools`:

```powershell
$maven = Get-ChildItem "C:\Tools" -Directory |
    Where-Object { $_.Name -match '^apache-maven-3\.9\.' } |
    Sort-Object Name -Descending |
    Select-Object -First 1

if (-not $maven) {
    throw "Maven 3.9 installation not found."
}

$env:MAVEN_HOME = $maven.FullName
$env:Path = "$env:MAVEN_HOME\bin;$env:Path"

[Environment]::SetEnvironmentVariable(
    "MAVEN_HOME",
    $maven.FullName,
    "User"
)

mvn -version
```

Add `%MAVEN_HOME%\bin` to your user PATH through Windows Environment Variables if it isn't already there.

### Expected verification

```text
Apache Maven 3.9.x
Java version: 21.x.x
Java home: C:\Program Files\Eclipse Adoptium\...
```

The key requirement is that **Maven is using Java 21**, not merely that Java 21 exists somewhere on your machine.

## 7. Create the LearnHub backend workspace

We will keep the backend next to the Angular frontend.

```powershell
cd $HOME

New-Item -ItemType Directory -Force angular-learning
Set-Location angular-learning

New-Item -ItemType Directory -Force learnhub-backend
Set-Location learnhub-backend

Get-Location
```

Expected directory:

```text
C:\Users\<your-windows-user>\angular-learning\learnhub-backend
```

We'll use this directory for Java and Spring Boot throughout the backend course.

## 8. Write your first LearnHub Java program

For Lesson 1, we'll build a small console application that models a course and prints a catalog. This is intentionally **plain Java**, not Spring Boot yet.

Create directories:

```powershell
cd $HOME\angular-learning\learnhub-backend

New-Item -ItemType Directory -Force src\main\java\com\learnhub\model
New-Item -ItemType Directory -Force src\main\java\com\learnhub
```

Your project will initially look like:

```text
learnhub-backend/
└── src/
    └── main/
        └── java/
            └── com/
                └── learnhub/
                    ├── Main.java
                    └── model/
                        └── Course.java
```

### Step 8.1 — Create the Course record

File: `src/main/java/com/learnhub/model/Course.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

import java.math.BigDecimal;

/**
 * Represents a course in the LearnHub catalog.
 *
 * Java records provide:
 * - A canonical constructor
 * - Accessor methods
 * - equals() and hashCode()
 * - toString()
 *
 * Records are useful for immutable data carriers.
 */
public record Course(
        long id,
        String title,
        String instructor,
        BigDecimal price,
        boolean free
) {

    /**
     * Validates course data at construction time.
     */
    public Course {
        if (id <= 0) {
            throw new IllegalArgumentException(
                    "Course ID must be positive."
            );
        }

        if (title == null || title.isBlank()) {
            throw new IllegalArgumentException(
                    "Course title is required."
            );
        }

        if (instructor == null || instructor.isBlank()) {
            throw new IllegalArgumentException(
                    "Instructor name is required."
            );
        }

        if (price == null || price.signum() < 0) {
            throw new IllegalArgumentException(
                    "Course price cannot be negative."
            );
        }

        if (free != (price.signum() == 0)) {
            throw new IllegalArgumentException(
                    "Free status must match the course price."
            );
        }
    }

    /**
     * Returns a human-readable price label.
     */
    public String priceLabel() {
        return free ? "FREE" : "$" + price.toPlainString();
    }
}
</CodeBlock>

### Why use `BigDecimal` for price?

Financial amounts should not normally be represented with binary floating-point values such as `double`, because decimal values can introduce precision problems.

For example:

```java
new BigDecimal("49.99")
```

is preferable to:

```java
new BigDecimal(49.99)
```

when the intention is an exact decimal amount.

In a production marketplace, we'll also introduce currency codes and enforce price rules in the database and payment layer.

### Step 8.2 — Create the Main class

File: `src/main/java/com/learnhub/Main.java`

<CodeBlock language="java" editable>
package com.learnhub;

import com.learnhub.model.Course;

import java.math.BigDecimal;
import java.util.List;

/**
 * LearnHub's first Java application.
 *
 * This console program demonstrates:
 * 1. Java records
 * 2. Collections
 * 3. Streams
 * 4. Basic validation
 * 5. Application entry points
 */
public final class Main {
    private Main() {
        // Utility-style entry point: no instances required.
    }

    public static void main(String[] args) {

        System.out.println("================================");
        System.out.println("      LearnHub Backend");
        System.out.println("================================");

        // Immutable list of sample courses.
        List<Course> courses = List.of(
                new Course(
                        1L,
                        "Angular Complete Guide",
                        "Alex Morgan",
                        new BigDecimal("49.99"),
                        false
                ),
                new Course(
                        2L,
                        "Python for Beginners",
                        "Sarah Lee",
                        BigDecimal.ZERO,
                        true
                ),
                new Course(
                        3L,
                        "UI/UX Design Masterclass",
                        "Emma Wilson",
                        new BigDecimal("29.99"),
                        false
                )
        );

        // Print the catalog.
        System.out.println("\nAvailable courses:");

        courses.forEach(course ->
                System.out.printf(
                        "#%d | %s | %s | %s%n",
                        course.id(),
                        course.title(),
                        course.instructor(),
                        course.priceLabel()
                )
        );

        // Count free courses using the Stream API.
        long freeCourseCount = courses.stream()
                .filter(Course::free)
                .count();

        System.out.println(
                "\nTotal courses: " + courses.size()
        );

        System.out.println(
                "Free courses: " + freeCourseCount
        );

        System.out.println("\nLearnHub is ready!");
    }
}
</CodeBlock>

Notice the difference between a record accessor and a traditional JavaBean getter:

```java
course.title();     // Record accessor
```

rather than:

```java
course.getTitle();  // Conventional JavaBean getter
```

We'll revisit records and JPA entities later. Records are particularly useful for API request and response DTOs, whereas JPA entities typically use regular classes.

## 9. Compile and run without Maven

Before introducing build automation, compile manually so the underlying Java toolchain is clear.

From the project root:

```powershell
cd $HOME\angular-learning\learnhub-backend

New-Item -ItemType Directory -Force out

javac --release 21 `
    -d out `
    src\main\java\com\learnhub\model\Course.java `
    src\main\java\com\learnhub\Main.java
```

Run:

```powershell
java -cp out com.learnhub.Main
```

Expected output:

```text
================================
      LearnHub Backend
================================

Available courses:
#1 | Angular Complete Guide | Alex Morgan | $49.99
#2 | Python for Beginners | Sarah Lee | FREE
#3 | UI/UX Design Masterclass | Emma Wilson | $29.99

Total courses: 3
Free courses: 1

LearnHub is ready!
```

### Understand the command

`javac --release 21` compiles for the Java 21 platform API and class-file version.

`-d out` places compiled class files in the `out` directory.

`java -cp out com.learnhub.Main` tells Java where the compiled classes are and which fully qualified class contains the entry point.

You can inspect the results:

```powershell
Get-ChildItem out -Recurse -Filter *.class
```

You should see compiled classes corresponding to `Main` and `Course`.

## 10. Convert the application into a Maven project

Now we'll introduce Maven, which will manage builds, dependencies, and automated tests throughout the Spring Boot course.

Create `pom.xml` in:

`$HOME\angular-learning\learnhub-backend\pom.xml`

<CodeBlock language="xml" editable>
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
           http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- Project identity -->
    <groupId>com.learnhub</groupId>
    <artifactId>learnhub-backend</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>LearnHub Backend</name>
    <description>Java and Spring Boot learning platform backend</description>

    <properties>
        <!-- Compile against the Java 21 platform -->
        <maven.compiler.release>21</maven.compiler.release>

        <!-- Consistent encoding -->
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Pin build and test plugin versions -->
        <maven.compiler.plugin.version>3.14.1</maven.compiler.plugin.version>
        <maven.surefire.plugin.version>3.5.3</maven.surefire.plugin.version>

        <!-- JUnit Jupiter 5 for this plain Java project -->
        <junit.version>5.12.2</junit.version>
    </properties>

    <dependencies>
        <!-- JUnit 5 API and test engine -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Java compilation -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven.compiler.plugin.version}</version>
            </plugin>

            <!-- JUnit 5 test execution -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven.surefire.plugin.version}</version>
            </plugin>

            <!-- Executable main class for java -jar -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.4.2</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.learnhub.Main</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
</CodeBlock>

This is deliberately a **plain Maven Java project**. In Lesson 5, we'll convert it into a Spring Boot application and adopt Spring Boot's managed dependency versions.

### Important Maven concepts

| Concept | Example | Purpose |
|---|---|---|
| Group ID | `com.learnhub` | Project namespace |
| Artifact ID | `learnhub-backend` | Artifact name |
| Version | `0.1.0-SNAPSHOT` | Development version |
| Dependency | `junit-jupiter` | External library |
| Plugin | `maven-compiler-plugin` | Build behavior |
| Scope | `test` | Dependency available for testing |
| Lifecycle | `package` | Executes earlier build phases and packages output |

### Step 10.1 — Compile

```powershell
mvn clean compile
```

Maven downloads required plugins and dependencies into the local Maven repository, normally:

```text
C:\Users\<your-windows-user>\.m2\repository
```

Compiled classes go into:

```text
target/classes/
```

### Step 10.2 — Package

```powershell
mvn clean package
```

Expected artifact:

```text
target/learnhub-backend-0.1.0-SNAPSHOT.jar
```

### Step 10.3 — Run the JAR

```powershell
java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar
```

You should see the same LearnHub catalog as before.

<box gap={2}>
  **Maven build lifecycle**

  <row align="center" gap={1} wrap="wrap">
    {#each ["validate","compile","test","package","verify","install"] as phase,i}
      <box background="surface-secondary" radius="md" padding={2}>
        <text size="xs" weight="medium">{phase}</text>
      </box>
      {#if i<5}<icon name="chevron-right" color="tertiary" size="xs"/>{/if}
    {/each}
  </row>
  <caption>Invoking a later lifecycle phase also runs the earlier required phases in that lifecycle. `clean` belongs to a separate lifecycle.</caption>
</box>

## 11. Write your first automated unit tests

Even though Lesson 21 will cover testing comprehensively, we'll introduce a basic test now to establish a good engineering habit.

Create:

```powershell
New-Item -ItemType Directory -Force `
    src\test\java\com\learnhub\model
```

File: `src/test/java/com/learnhub/model/CourseTest.java`

<CodeBlock language="java" editable>
package com.learnhub.model;

import org.junit.jupiter.api.Test;

import java.math.BigDecimal;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for LearnHub's Course record.
 */
class CourseTest {

    @Test
    void freeCourseShouldHaveFreePriceLabel() {

        Course course = new Course(
                2L,
                "Python for Beginners",
                "Sarah Lee",
                BigDecimal.ZERO,
                true
        );

        assertEquals("FREE", course.priceLabel());
        assertTrue(course.free());
    }

    @Test
    void paidCourseShouldDisplayPrice() {

        Course course = new Course(
                1L,
                "Angular Complete Guide",
                "Alex Morgan",
                new BigDecimal("49.99"),
                false
        );

        assertEquals("$49.99", course.priceLabel());
    }

    @Test
    void courseShouldRejectNegativePrice() {

        assertThrows(
                IllegalArgumentException.class,
                () -> new Course(
                        3L,
                        "Invalid Course",
                        "Test Instructor",
                        new BigDecimal("-10.00"),
                        false
                )
        );
    }

    @Test
    void courseShouldRejectInvalidId() {

        assertThrows(
                IllegalArgumentException.class,
                () -> new Course(
                        0L,
                        "Invalid Course",
                        "Test Instructor",
                        BigDecimal.ZERO,
                        true
                )
        );
    }

    @Test
    void freeStatusMustMatchPrice() {

        assertThrows(
                IllegalArgumentException.class,
                () -> new Course(
                        4L,
                        "Inconsistent Course",
                        "Test Instructor",
                        new BigDecimal("19.99"),
                        true
                )
        );
    }
}
</CodeBlock>

Run the tests:

```powershell
mvn test
```

Expected summary:

```text
Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

This establishes a useful distinction:

- `mvn compile` checks whether production code compiles.
- `mvn test` compiles and runs tests.
- `mvn package` runs the required earlier build phases and creates a JAR.
- `mvn clean verify` is a good broader local verification command.

For a fast edit-test cycle:

```powershell
mvn -q test
```

For full verification:

```powershell
mvn clean verify
```

## 12. Import the Maven project into IntelliJ IDEA

Open IntelliJ IDEA, select **Open**, and choose:

```text
C:\Users\<your-windows-user>\angular-learning\learnhub-backend
```

IntelliJ should detect `pom.xml` and offer to import or load the Maven project.

Verify these settings:

| Setting | Expected value |
|---|---|
| Project SDK | JDK 21 |
| Language level | Java 21 |
| Maven project | `learnhub-backend` |
| Source directory | `src/main/java` |
| Test directory | `src/test/java` |
| Build file | `pom.xml` |

Open `Main.java`, click the run icon beside `main()`, and verify the output.

Next, open `CourseTest.java` and run the test class. All five tests should pass.

### Useful IntelliJ shortcuts

| Action | Windows shortcut |
|---|---|
| Search everywhere | Double Shift |
| Find class | Ctrl+N |
| Find file | Ctrl+Shift+N |
| Reformat code | Ctrl+Alt+L |
| Rename symbol | Shift+F6 |
| Quick fix | Alt+Enter |
| Run | Shift+F10 |
| Debug | Shift+F9 |

## 13. Prepare Git version control

Since you're maintaining a learning journal, it's useful to keep the actual backend source code under version control as well.

Create a `.gitignore` in the backend project root:

```gitignore
# Maven build output
target/

# Manual Java compilation output
out/

# IntelliJ IDEA local files
.idea/
*.iml

# Local environment and secrets
.env
.env.*
!.env.example

# Operating system files
.DS_Store
Thumbs.db

# Log files
*.log
```

Initialize a local repository if this is a new standalone project:

```powershell
cd $HOME\angular-learning\learnhub-backend

git init
git add .
git commit -m "Initialize LearnHub Java 21 Maven backend"```

If Git asks for your name and email, configure them using your preferred Git identity before committing.

This creates a **local commit only**. It does not upload the backend source code to GitHub. We'll decide how to organize the backend repository and its learning journal separately.

## 14. Common Windows setup problems

<box gap={3}>
  <box gap={1}>
    **Problem: `java` is not recognized**

    Check:

    ```powershell
    $env:JAVA_HOME
    $env:Path -split ';'
    ```

    Ensure the JDK's `bin` directory is on PATH, then reopen PowerShell.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: `javac` is missing but `java` works**

    You may be using an unexpected Java runtime or PATH entry. Check:

    ```powershell
    where.exe java
    where.exe javac
    ```

    Confirm that both resolve to the intended JDK 21 installation.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Maven uses the wrong Java version**

    Run:

    ```powershell
    mvn -version
    $env:JAVA_HOME
    ```

    Ensure `JAVA_HOME` points to JDK 21 and restart the terminal.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: Maven cannot download dependencies**

    Check internet connectivity, Maven repository settings, and any corporate proxy configuration.

    ```powershell
    mvn -U clean test
    ```

    `-U` forces Maven to check for updated snapshots and missing releases. It will not fix a network or proxy outage by itself.
  </box>
  <divider color="subtle"/>
  <box gap={1}>
    **Problem: `UnsupportedClassVersionError`**

    The application was compiled for a newer Java version than the runtime executing it.

    Compare:

    ```powershell
    java -version
    javac -version
    mvn -version
    ```

    Use Java 21 consistently for this project.
  </box>
</box>

## 15. Hands-on exercises

<box gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md">
      <icon name="code" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise A — Add a course category**

      Extend the `Course` record with a `String category` field. Validate that it is not blank, update all constructor calls, and print the category in the catalog.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md">
      <icon name="list-filter" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise B — Filter free courses**

      Use the Stream API to return a list of free courses. Print their titles and add a unit test for the filtering logic. Consider extracting catalog operations into a separate class rather than testing only console output.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md">
      <icon name="calculator" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise C — Calculate catalog value**

      Use `BigDecimal` and `reduce()` to calculate the sum of all course prices. The three initial courses should total **$79.98**.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md">
      <icon name="shield-alert" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise D — Add validation tests**

      Add tests for blank titles, null prices, negative prices, and inconsistent free/paid status. Verify that invalid objects cannot be constructed.
    </box>
  </row>
  <divider color="subtle"/>
  <row align="start" gap={3}>
    <box background="surface-secondary" padding={2} radius="md">
      <icon name="package" size="lg"/>
    </box>
    <box flex="1" gap={1}>
      **Exercise E — Build from a clean checkout**

      Delete generated `out/` and `target/` directories, then run:

      ```powershell
      mvn clean verify
      java -jar target\learnhub-backend-0.1.0-SNAPSHOT.jar
      ```

      Confirm the program works without relying on IntelliJ's internal build output.
    </box>
  </row>
</box>

## 16. Lesson 1 knowledge quiz

<LearningQuiz
  title="Java 21, JVM, Maven, and LearnHub Setup"
  nextQuizQuery="Give me 5 new intermediate-to-advanced questions about Java 21 compilation, JVM runtime behavior, Maven lifecycles, BigDecimal, records, and JUnit, without repeating the Lesson 1 quiz."
  questions={[
    {
      "id":"q1",
      "type":"single_select",
      "question":"A Java application was compiled for Java 21 but is launched using a Java 17 runtime. What is the likely result?",
      "options":[
        {"value":"A","label":"The runtime recompiles the classes automatically","feedback":"The Java launcher does not automatically recompile class files for an older platform."},
        {"value":"B","label":"An UnsupportedClassVersionError occurs","feedback":"Java 17 cannot load class files targeting the newer Java 21 class-file version."},
        {"value":"C","label":"Maven silently downgrades the bytecode","feedback":"Maven is not involved when launching the already compiled class files with Java."},
        {"value":"D","label":"The program runs using Java 17 compatibility mode","feedback":"An older JVM does not provide a general mode for executing newer class-file versions."}
      ],
      "correctValues":["B"],
      "hint":"Think about which direction Java runtime compatibility generally supports."
    },
    {
      "id":"q2",
      "type":"single_select",
      "question":"Which Maven command runs the tests and packages the LearnHub application into a JAR using the standard lifecycle?",
      "options":[
        {"value":"A","label":"mvn compile","feedback":"The compile phase does not reach the test and package phases."},
        {"value":"B","label":"mvn test","feedback":"The test phase runs tests but does not execute the later package phase."},
        {"value":"C","label":"mvn package","feedback":"The package phase executes earlier required phases, including test, before packaging."},
        {"value":"D","label":"mvn validate","feedback":"The validate phase occurs before compilation and testing."}
      ],
      "correctValues":["C"],
      "hint":"Maven phases form an ordered build lifecycle."
    },
    {
      "id":"q3",
      "type":"multi_select",
      "question":"Which statements about the LearnHub Course record are correct? Select all that apply.",
      "options":[
        {"value":"A","label":"A record generates accessor methods for its components","feedback":"Record components receive accessor methods such as title() and price()."},
        {"value":"B","label":"A record can validate arguments in a compact constructor","feedback":"Compact constructors can check arguments and reject invalid record instances."},
        {"value":"C","label":"A record automatically guarantees that every referenced object is deeply immutable","feedback":"Records are shallowly immutable; referenced mutable objects can still change."},
        {"value":"D","label":"A record automatically provides equals and hashCode implementations","feedback":"Java generates component-based equality and hash code behavior for records."}
      ],
      "correctValues":["A","B","D"],
      "hint":"Separate features generated by records from the mutability of objects they reference."
    },
    {
      "id":"q4",
      "type":"true_false",
      "question":"The Java command `javac --release 21` configures compilation against the Java 21 platform APIs as well as the corresponding class-file target.",
      "options":[
        {"value":"true","label":"True","feedback":"The release option constrains both the Java platform APIs available at compile time and the generated class-file version."},
        {"value":"false","label":"False","feedback":"The release option does more than select bytecode version; it also restricts the supported platform API surface."}
      ],
      "correctValues":["true"],
      "hint":"Consider the difference between selecting a bytecode target and selecting a complete Java platform release."
    },
    {
      "id":"q5",
      "type":"true_false",
      "question":"Setting JAVA_HOME to a JDK's bin directory is the standard configuration expected by Maven.",
      "options":[
        {"value":"true","label":"True","feedback":"Maven expects JAVA_HOME to identify the JDK installation root, not its bin subdirectory."},
        {"value":"false","label":"False","feedback":"JAVA_HOME should point to the JDK root; the bin directory is added separately to PATH."}
      ],
      "correctValues":["false"],
      "hint":"Think about the directory structure of a JDK installation."
    }
  ]}
/>

## 17. Lesson completion checklist

{@body const checks=[{group:"Environment",items:["JDK 21 is installed","java and javac report version 21","JAVA_HOME points to the JDK root","Maven runs with Java 21","IntelliJ IDEA recognizes JDK 21"]},{group:"Java project",items:["LearnHub backend folder exists","Course.java record is implemented","Main.java prints the catalog","Manual javac compilation works","Maven project imports into IntelliJ"]},{group:"Build and testing",items:["mvn clean compile succeeds","All five JUnit tests pass","mvn clean verify succeeds","The packaged JAR runs","The project has a .gitignore and a local Git commit"]}]}
{@body const [done,setDone]=DIL.useState({})}
{@body const count=Object.values(done).filter(Boolean).length}
{@body const total=checks.reduce((sum,g)=>sum+g.items.length,0)}
<box border radius="xl" padding={3} gap={3}>
  <row align="center" justify="between">
    <box gap={1}>
      **Lesson 1 progress**
      <text size="xs" color="secondary">Mark tasks as you verify them on your computer.</text>
    </box>
    <title size="lg" tabularNums>{count}/{total}</title>
  </row>
  <box background="surface-tertiary" height="8px" radius="full" clip>
    <box background="rgba(109,40,217,0.85)" width={`${100*count/total}%`} height="100%" radius="full"/>
  </box>
  {#each checks as group,i}
    <box gap={2} key={i}>
      <text weight="medium">{group.group}</text>
      {#each group.items as item,j}
        <checkbox key={`${i}-${j}`} checked={!!done[`${i}-${j}`]} onChange={v=>setDone(old=>({...old,[`${i}-${j}`]:v}))}>{item}</checkbox>
      {/each}
    </box>
  {/each}
  {#if count===total}
    <row align="center" gap={2}>
      <icon name="check-circle" color="success"/>
      **Lesson 1 verification checklist completed.**
    </row>
  {/if}
</box>

## 18. What's next?

<row align="start" gap={3}>
  <box background="surface-secondary" padding={3} radius="lg">
    <icon name="arrow-right" size="xl"/>
  </box>
  <box flex="1" gap={2}>
    **Lesson 2 — Java Fundamentals for Backend Development**

    We'll review Java types, control flow, methods, records, `BigDecimal`, input validation, and domain modeling through practical LearnHub examples. Since you have Java experience, we'll focus on backend-oriented decisions and common implementation mistakes rather than spending excessive time on elementary syntax.

    <button block onClick={()=>GenUI.issueNewTurn("Start LearnHub Java and Spring Boot Backend Lesson 2: Java Fundamentals for Backend Development, with experienced-Java pacing. Use Windows PowerShell commands, complete file paths, documented LearnHub code, exercises, quiz, and checklist. Continue from the Lesson 1 Maven project.")}>Start Lesson 2 <icon name="arrow-right" inline/></button>
  </box>
</row>

**Lesson 1 deliverable:** A Java 21 Maven project containing a validated `Course` record, a working LearnHub console catalog, and automated JUnit tests. This will become the foundation of your Spring Boot backend.