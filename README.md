# Artifact Repository Manager using Nexus

**Project:** Run Nexus on a DigitalOcean Droplet and publish Java artifacts to it

**Technologies used:** Sonatype Nexus Repository, DigitalOcean, Linux (Ubuntu), Java 17, Gradle, Maven

**Key project tasks and activities:**

- Installed and configured Nexus from scratch on a cloud server, running as a dedicated non-root Linux user.
- Created a dedicated Nexus user and role with only the permissions needed to publish artifacts (least privilege).
- Built a Java Gradle project into a JAR and published it to Nexus.
- Built a Java Maven project into a JAR and deployed it to Nexus.

---

## Description

#### What is an artifact repository?

A CI pipeline produces **build artifacts** which is: *an application packaged into files such as JAR, WAR, ZIP or a Docker image*. **An artifact repository** stores those artifacts centrally so every environment and team pulls the exact same version rather than rebuilding it or passing files around.

#### What is an artifact repository manager?

Instead of a separate store for every format, a **repository manager** hosts many repository types in one system. That matters in companies that build Java, .NET and Docker images side by side.

**Public repository managers** (for example Maven Central) host open-source libraries consumed as dependencies.
**Private repository managers** (for example Nexus) host a company's internal artifacts, with access control.

###### Why Nexus fits into CI/CD

Nexus sits in the delivery path: builds publish to it and the deployments pull from it. Features that make it suitable for automation include multi-format support (Maven, npm, Docker, NuGet), a REST API, user tokens for non-interactive auth, LDAP integration, cleanup policies and backup and restore.

### Prerequisites

A [DigitalOcean ](https://www.digitalocean.com/) account and an **SSH client** (`ssh`, `ssh-keygen`, `scp`)
**Java 17**, **Gradle** and **Maven** installed locally
**Example projects in this repo:**

- [java-gradle-app](./java-app/) : Gradle / Spring Boot
- [java-maven-app](./java-maven-app): Maven / Spring Boot

**Server used**: Ubuntu Droplet, 4 vCPU / 8 GB RAM, London region. Nexus needs at least 4 GB of RAM, and below that it struggles to start. Nexus version: Sonatype Nexus Repository 3.96.4-01 (Community)

---

## Installing and Running Nexus on a Cloud Server

### Part 1: Server, Java and Nexus

###### 1. Created the Droplet. I provisioned an Ubuntu LTS Droplet with SSH key authentication, sized for Nexus.

###### 2. Opened SSH (port 22) in the DigitalOcean Cloud Firewall for administration.

###### 3. Installed Java on the server. Nexus requires a specific Java version for each release, so I checked Sonatype's requirements for 3.96 and installed the matching JDK:

```bash
java -version
```

###### 4. Downloaded and extracted Nexus into /opt, which produces two folders:

`nexus-3.96.4-01/`: the application
`sonatype-work/`: data, configuration and stored artifacts

###### 5. Created a dedicated `nexus` Linux user. Services should not run as root. If the service is compromised, the damage is limited to what that user can access.
   
   ### Part 2: Ownership, starting Nexus and network access

###### 6. Gave the nexus user ownership of both folders:

```bash
chown -R nexus:nexus /opt/nexus-3.96.4-01
chown -R nexus:nexus /opt/sonatype-work
```

###### 7. Configured Nexus to run as `nexus` and started it:

```bash
su - nexus
/opt/nexus-3.96.4-01/bin/nexus start
```
###### 8. Opened port 8081 in the firewall. Nexus serves its UI and repositories on 8081, so SSH (22) alone doesn't make it reachable from a browser.

###### 9. Accessed the UI at `http://<droplet-ip>:8081` and signed in as `admin` using the initial password from `sonatype-work/nexus3/admin.password`, then changed it.

#### Repository Types

| Type | Purpose |
|---|---|
| **Hosted** | Stores artifacts we publish ourselves (used here: `maven-snapshots`) |
| **Proxy** | Caches an upstream repository such as Maven Central |
| **Group** | Combines several repositories behind one URL |

Snapshots vs releases: versions ending in `-SNAPSHOT` (for example 1.0-SNAPSHOT) are in-progress builds and go to `maven-snapshots`. Final versions go to `maven-releases`.

### Creating a Nexus User with Deployment Permissions

**Goal**: avoid publishing as `admin` by applying least privilege.

- Created a role, [`nx-java`], with the privilege `nx-repository-view-maven2-maven-snapshots-*`, which allows uploading to the snapshot repository.
- Created a Nexus user, `ryan` and assigned it that role.
- Used this user's credentials in Gradle and Maven which are kept outside version control.

###### Note: Nexus users are separate from Linux users. Gradle and Maven authenticate to the Nexus application over HTTP, not to the server's operating system.

---
## Publishing Artifacts to Nexus with Gradle

The project in `java-gradle-app` uses the `maven-publish` plugin. In build.gradle, I pointed the publishing repository at the hosted snapshot repository:

```xml
repositories {
    maven {
        name = 'nexus'
        url = "http://<nexus-host>:8081/repository/maven-snapshots/"
        allowInsecureProtocol = true   // lab only: HTTP, not HTTPS
        credentials {
            username = project.repoUser
            password = project.repoPassword
        }
    }
}
```
Credentials live in gradle.properties, which is `git-ignored` so the password is never committed:

```xml
repoUser=<nexus-user>
repoPassword=<nexus-password>
```
Then I built and published:

```bash
gradle build
gradle publish
```
I confirmed the JAR appeared under **Browse → maven-snapshots** in Nexus.

## Publishing Artifacts to Nexus with Maven

The project in `java-maven-app` declares the target repository in `pom.xml` under `<distributionManagement>`, with the id `nexus-snapshots`.

Maven reads credentials from `~/.m2/settings.xml` (outside the project), matched by that same `<id>`:

```xml
<settings>
  <servers>
    <server>
      <id>nexus-snapshots</id>
      <username>NEXUS_USER</username>
      <password>NEXUS_PASSWORD</password>
    </server>
  </servers>
</settings>
```
Then I deployed:

```bash
mvn package
mvn deploy
```

###### [Confirm: the artifact appeared under maven-snapshots in Nexus.]

## Challenges and Fixes

| Problem | How I investigated | Root cause and fix |
|---|---|---|
| `gradle publish` failed with **"Broken pipe"** | Ran `gradle publish --info`: the metadata GET succeeded, but every upload PUT failed. Then I ran a `curl` upload with the Nexus user, which returned **HTTP 400 (invalid path)**, proving that authentication and permissions were fine. | `gradle.properties` still contained the course's **placeholder credentials**, so Nexus rejected the upload mid-transfer, which surfaced as "Broken pipe". I replaced them with my Nexus user's credentials. |
| `mvn package` failed with **"Non-parseable settings"** | The error pointed to line 9 of `~/.m2/settings.xml` | A typo in a closing tag (`</servers.>`). I corrected it to `</servers>`. |

**Lesson:** a misleading error message ("Broken pipe") can hide an authentication problem. Testing each layer separately (network, then Nexus auth, then build tool config) isolated the cause quickly.

#### Key Concepts
- **Component vs asset**: a component is the logical item (for example `my-app 1.0-SNAPSHOT`). Assets are the physical files that belong to it (the JAR, the POM, checksums).
- **REST API**: Nexus can be automated over `/service/rest/v1/`, for example listing a repository's components:

```bash
  curl -u <user> -X GET "http://<nexus-host>:8081/service/rest/v1/components?repository=maven-snapshots"
```
- **Cleanup policies**: rules (for example "delete snapshots older than 30 days") run on a schedule to reclaim disk space.

##### Security Considerations
###### - Nexus runs as a dedicated nexus Linux user, not root.
###### - Artifacts are published by a least-privilege Nexus user, not admin.
###### - Credentials are never committed: gradle.properties is `git-ignored`, and Maven credentials live in `~/.m2/settings.xml` outside the repo.
###### - Limitation: this lab uses HTTP (`allowInsecureProtocol = true`), so credentials travel unencrypted.

#### Skills Demonstrated

`Artifact management` · `Sonatype Nexus` · `Gradle` · `Maven` · `Linux server administration` · `Least-privilege access` · `Secrets handling` · `Troubleshooting & debugging` · `Cloud (IaaS)`

## References

- Sonatype **Nexus Repository** documentation: [help.sonatype.com](https://help.sonatype.com/) (installation, repositories, security, REST API)
- DigitalOcean: [Droplets](https://docs.digitalocean.com/products/droplets/), [Cloud Firewalls](https://docs.digitalocean.com/networking/firewalls/)

