# Publishing to Central Portal using GitHub Actions

This guide outlines how to publish Maven-based Eclipse projects directly to [Central Portal](https://central.sonatype.org) using GitHub Actions.

## Prerequisites

* Account and namespace already claimed. [Open a HelpDesk Issue](https://gitlab.eclipse.org/eclipsefdn/helpdesk/-/issues/new)

* Publishing credentials stored as GitHub secrets

This is the list of secrets needed to deploy to central declared in GitHub organization as secrets.

* `CENTRAL_SONATYPE_TOKEN_USERNAME`
* `CENTRAL_SONATYPE_TOKEN_PASSWORD`
* `GPG_KEY_ID`
* `GPG_PASSPHRASE`
* `GPG_PRIVATE_KEY`

Otterdog configuration: e.g: https://github.com/eclipse-cbi/.eclipsefdn/blob/main/otterdog/eclipse-cbi.jsonnet

```js
secrets+: [
    orgs.newOrgSecret('CENTRAL_SONATYPE_TOKEN_PASSWORD') {
      value: "pass:bots/<project_id>/central.sonatype.org/token-password",
    },
    orgs.newOrgSecret('CENTRAL_SONATYPE_TOKEN_USERNAME') {
      value: "pass:bots/<project_id>/central.sonatype.org/token-username",
    },
    orgs.newOrgSecret('GPG_KEY_ID') {
      value: "pass:bots/<project_id>/gpg/key_id",
    },
    orgs.newOrgSecret('GPG_PASSPHRASE') {
      value: "pass:bots/<project_id>/gpg/passphrase",
    },
    orgs.newOrgSecret('GPG_PRIVATE_KEY') {
      value: "pass:bots/<project_id>/gpg/secret-subkeys.asc",
    },
  ],
```

## Maven Configuration (`pom.xml`)

### Central Publishing Plugin

Docs: [Automatic Publishing](https://central.sonatype.org/publish/publish-portal-maven/#automatic-publishing)

Add the plugin to your `pom.xml`:


```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.sonatype.central</groupId>
      <artifactId>central-publishing-maven-plugin</artifactId>
      <version>0.7.0</version>
      <extensions>true</extensions>
      <configuration>
        <publishingServerId>central</publishingServerId>
        <autoPublish>true</autoPublish>
        <waitUntil>published</waitUntil>
        <centralSnapshotsUrl>https://central.sonatype.com/repository/maven-snapshots/</centralSnapshotsUrl>
      </configuration>
    </plugin>
  </plugins>
</build>
```

### GPG Signing in GitHub Actions

1. Add the Maven GPG plugin to your `pom.xml`:

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-gpg-plugin</artifactId>
      <version>1.5</version>
      <executions>
        <execution>
          <id>sign-artifacts</id>
          <phase>verify</phase>
          <goals>
            <goal>sign</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

## GitHub Actions Workflow Example

Example of snapshot deployment:

{% raw %}
```yaml
name: Publish Snapshot package to the Maven Central Repository
on:
  push:
    branches:
      - main
      - develop

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Maven Central Repository
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: central
          server-username: MAVEN_USERNAME
          server-password: MAVEN_PASSWORD
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: MAVEN_GPG_PASSPHRASE
      - name: Publish package
        run: mvn --batch-mode deploy -DskipTests
        env:
          MAVEN_USERNAME: ${{ secrets.CENTRAL_SONATYPE_TOKEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.CENTRAL_SONATYPE_TOKEN_PASSWORD }}
          MAVEN_GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```
{% endraw %}

Publish release depends on project strategy, this is an example:

{% raw %}
```yaml
name: Publish Release to Maven Central

on:
  release:
    types: [published] 

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Set up Java and Maven Central credentials
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: central
          server-username: MAVEN_USERNAME
          server-password: MAVEN_PASSWORD
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: MAVEN_GPG_PASSPHRASE

      - name: Set project version from tag
        run: mvn --batch-mode versions:set -DnewVersion=${{ github.event.release.tag_name }} -DgenerateBackupPoms=false

      - name: Publish to Central
        run: mvn --batch-mode deploy -DskipTests
        env:
          MAVEN_USERNAME: ${{ secrets.CENTRAL_SONATYPE_TOKEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.CENTRAL_SONATYPE_TOKEN_PASSWORD }}
          MAVEN_GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
```
{% endraw %}

## Snapshot Publishing (Optional)

If publishing **snapshots** to Central, add this to the plugin config:

````xml
<centralSnapshotsUrl>https://central.sonatype.com/repository/maven-snapshots/</centralSnapshotsUrl>
````

Or use the `<distributionManagement>` in pom.xml.

Doc: [Snapshot publishing](https://central.sonatype.org/publish/publish-portal-snapshots/)

NOTE: The snapshot feature is enabled by default for all namespaces on Central Portal.

## Best Practices

- Use `central-publishing-maven-plugin` with `autoPublish` and `waitUntil`
- Sign all artifacts with GPG (even on snapshots)
- Triggered release on release published event. 

## Resources

- [Central Portal](https://central.sonatype.com)
- [Publishing with Central Plugin](https://central.sonatype.org/publish/publish-portal-maven/)
- [GPG Signing Guide](https://central.sonatype.org/publish/publish-maven/#gpg-signed-components)