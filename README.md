# Customer Management System

A console-based Java order and payment management application, built as an end-to-end DevOps
exercise: version control with Git Flow, build automation with Maven, unit testing with JUnit 5,
continuous integration with Jenkins, containerisation with Docker, and cloud deployment on AWS EC2.

**Course:** MACSE639 — DevOps (ETH + ELA) · Winter Semester 2025–26
**Institution:** School of Computer Science and Engineering (SCOPE), VIT Vellore
**Faculty:** Sudhakar P B · **Group 10, Set B**

---

## Table of Contents

- [Domain Model](#domain-model)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Maven Lifecycle](#maven-lifecycle)
- [Testing](#testing)
- [Branching Strategy](#branching-strategy)
- [CI/CD Pipeline](#cicd-pipeline)
- [Docker](#docker)
- [Cloud Deployment (AWS EC2)](#cloud-deployment-aws-ec2)
- [Sample Output](#sample-output)
- [Team](#team)

---

## Domain Model

All classes live in the `customermgmt` package.

| Class | Responsibility |
| --- | --- |
| `Customer` | Holds `name`, `address` and a list of orders; `addOrder()` |
| `Order` | Holds date, status, order details and payments; `calcSubTotal()`, `calcTax()`, `calcTotal()`, `calcTotalWeight()` |
| `OrderDetail` | Links an `Item` to a quantity and tax status; `calcSubTotal()`, `calcWeight()`, `calcTax()` |
| `Item` | Product data — `description`, `pricePerUnit`, `taxRate`, `shippingWeight`, `stockQuantity`; `getPriceForQuantity()`, `getTax()`, `inStock()` |
| `Payment` | Abstract base class holding `amount`; never instantiated directly |
| `Cash` | `Payment` subclass with `cashTendered`; `getChange()` |
| `Check` | `Payment` subclass with `name`, `bankID`; `authorized()` requires a 9-digit bank ID |
| `Credit` | `Payment` subclass with `name`, `type`, `expDate`; `authorized()` checks the card has not expired |
| `Main` | Driver — builds sample items, a customer, an order and all three payment types, then prints results |

```
Customer 1 ──── * Order 1 ──── * OrderDetail * ──── 1 Item
                     │
                     └──── * Payment ◁── Cash | Check | Credit
```

## Tech Stack

| Concern | Tool |
| --- | --- |
| Language / runtime | Java 17 |
| Build | Apache Maven (`pom.xml`, packaging `jar`) |
| Testing | JUnit 5 (Jupiter) 5.10.0 + Surefire |
| IDE | Apache NetBeans 20 |
| SCM | Git + GitHub (Git Flow) |
| CI/CD | Jenkins (declarative pipeline) |
| Container | Docker (multi-stage build) |
| Cloud | AWS EC2 (Ubuntu) + Docker Hub |

## Project Structure

```
customer-management/
├── Dockerfile
├── Jenkinsfile
├── pom.xml
├── .gitignore                       # target/, nbproject/private/, *.class, build/, dist/
└── src/
    ├── main/java/customermgmt/
    │   ├── Customer.java
    │   ├── Order.java
    │   ├── OrderDetail.java
    │   ├── Item.java
    │   ├── Payment.java
    │   ├── Cash.java
    │   ├── Check.java
    │   ├── Credit.java
    │   └── Main.java
    └── test/java/customermgmt/
        └── CustomerManagementTest.java
```

## Getting Started

### Prerequisites

- JDK 17
- Apache Maven 3.9+
- Docker (optional, for containerised runs)

### Clone, build and run

```bash
git clone https://github.com/AryaS01/customer-management.git
cd customer-management

mvn clean package
java -jar target/customer-management-1.0.0.jar
```

Or run through the exec plugin (main class is declared in `pom.xml`):

```bash
mvn exec:java
```

### pom.xml essentials

```xml
<groupId>com.customermgmt</groupId>
<artifactId>customer-management</artifactId>
<version>1.0.0</version>
<packaging>jar</packaging>

<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <maven.compiler.release>17</maven.compiler.release>
  <exec.mainClass>customermgmt.Main</exec.mainClass>
</properties>

<dependencies>
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
  </dependency>
</dependencies>
```

## Maven Lifecycle

| Phase | Command | What it does |
| --- | --- | --- |
| validate | `mvn validate` | Validates `pom.xml` and project structure |
| compile | `mvn compile` | Compiles `src/main/java` into `target/classes` |
| test | `mvn test` | Runs JUnit tests via Surefire |
| package | `mvn package` | Produces `customer-management-1.0.0.jar` |
| install | `mvn install` | Installs the JAR into the local repository |
| deploy | `mvn deploy` | Uploads the artifact to a remote repository |

## Testing

Ten JUnit 5 test cases in `CustomerManagementTest.java` cover the business logic, with a
`@BeforeEach` fixture that builds a laptop item, a mouse item, two order details and an order:

- `testOrderSubTotal` — subtotal across order details
- `testOrderTax` — tax applied only to taxable items
- `testOrderTotal` — subtotal plus tax
- `testItemInStock` — stock availability check
- `testCashChange` — change returned by `Cash.getChange()`
- `testCheckAuthorized` — 9-digit bank ID validation
- `testCreditAuthorization` — expiry-date check
- `testCustomerOrderAssociation` — orders attach to the customer
- `testOrderStatusUpdate` — status transitions
- `testOrderWeight` — total shipping weight

```bash
mvn test
```

Surefire reports are written to `target/surefire-reports/`.

## Branching Strategy

Git Flow: every member works on a dedicated feature branch and merges into `develop` via Pull
Request; `develop` merges into `main` for releases.

| Branch | Purpose |
| --- | --- |
| `main` | Production-ready code only; protected |
| `develop` | Integration branch — all feature branches merge here |
| `feature/customer-model` | `Customer.java` |
| `feature/order-model` | `Order.java`, `OrderDetail.java` |
| `feature/item-model` | `Item.java` |
| `feature/payment-model` | `Payment`, `Cash`, `Check`, `Credit` |
| `feature/unit-tests` | `CustomerManagementTest.java` |

Release tag:

| Tag | Description |
| --- | --- |
| `v1.0.0` | First stable release — all classes complete, CI green |

```bash
git checkout develop
git checkout -b feature/item-model
# ... work, commit ...
git push -u origin feature/item-model
# open a Pull Request into develop on GitHub

# tagging a release
git checkout main
git merge --no-ff develop
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin main --tags
```

## CI/CD Pipeline

A Jenkins job (`customer-mgmt-pipeline`) of type *Pipeline* is configured with **Pipeline script
from SCM**, pointing at the GitHub repository. It triggers on every push (webhook or polling) and
runs Checkout → Build → Test → Package → Docker Build.

```groovy
pipeline {
    agent any
    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/AryaS01/customer-management.git'
            }
        }
        stage('Build') {
            steps { sh 'mvn clean compile' }
        }
        stage('Unit Test') {
            steps { sh 'mvn test' }
            post {
                always { junit 'target/surefire-reports/*.xml' }
            }
        }
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t customer-mgmt:${BUILD_NUMBER} .'
            }
        }
    }
    post {
        failure { echo 'Build failed!' }
        success { echo 'Pipeline completed successfully!' }
    }
}
```

## Docker

Multi-stage build — Maven compiles the JAR in the first stage, and only the runtime artifact is
copied into a slim JRE image.

```dockerfile
FROM maven:3.9.6-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/customer-management-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Build and run locally:

```bash
docker build -t customer-app .
docker run --rm customer-app
```

Publish to Docker Hub:

```bash
docker tag customer-app lavanyasingh1812/customer-app
docker push lavanyasingh1812/customer-app
```

## Cloud Deployment (AWS EC2)

Launch an Ubuntu EC2 instance with an appropriate security group, then:

```bash
# 1. connect
ssh -i "customer-key.pem" ubuntu@<public-ip>

# 2. install Docker
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker

# 3. pull the published image
docker pull lavanyasingh1812/customer-app

# 4. run it
docker run -d -p 9090:8080 lavanyasingh1812/customer-app

# 5. verify — the app is console-based, so check the logs
docker logs <container-id>
```

**Note:** the application is a console program. It prints its output and exits rather than serving
HTTP, so the port mapping is not exercised — verification is done through `docker logs`.

## Sample Output

```
=== Items ===
Item{description='Laptop', price=999.99, inStock=true}
Item{description='Mouse', price=29.99, inStock=true}

=== Order Summary ===
SubTotal : 2089.95
Tax      : 376.19
Total    : 2466.14
Weight   : 5.90 kg

=== Payment ===
Cash{amount=2466.141, tendered=2200.0, change=-266.1411}
Check{name='Alice', bankID='123456789', authorized=true}
Credit{name='Alice', type='VISA', authorized=true}

=== Customer ===
Customer{name='Alice', address='123 Main St, Chennai'}
Orders: 1
```

## Team

| Name | Registration No. |
| --- | --- |
| Aryalekshmi S 
| Lavanya Singh 
| Edamalapati Saranya 

Submitted 30 March 2026.
