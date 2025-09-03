# Upload build artifacts to download.eclipse.org

This guide is intended to help with uploading build artifacts to a project's directory on download.eclipse.org using GitHub actions

## Prerequisites

* Publishing credentials stored as GitHub secrets
  * `SCP_HOST`
  * `SCP_USERNAME`
  * `SCP_KEY`

Please open a new [HelpDesk issue](https://gitlab.eclipse.org/eclipsefdn/helpdesk/-/issues/new) to request the creation of the SCP credentials.

Otterdog configuration: e.g.: https://github.com/eclipse-cbi/.eclipsefdn/blob/main/otterdog/eclipse-cbi.jsonnet#L267-L277

```
      secrets+: [
        orgs.newRepoSecret('SCP_KEY') {
          value: "pass:bots/technology.cbi/projects-storage.eclipse.org/id_ed25519",
        },
        orgs.newRepoSecret('SCP_PASSPHRASE') {
          value: "pass:bots/technology.cbi/projects-storage.eclipse.org/id_ed25519.passphrase",
        },
        orgs.newRepoSecret('SCP_USERNAME') {
          value: "pass:bots/technology.cbi/projects-storage.eclipse.org/username",
        },
      ],
```

## GitHub action workflow

{% raw %}
```yaml
name: Publish to downloads.eclipse.org

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@f43a0e5ff2bd294095638e18286ca9a3d1956744 # v3.6.0
    - name: Set up JDK 17
      uses: actions/setup-java@0ab4596768b603586c0de567f2430c30f5b0d2b0 # v3.13.0
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    - name: Setup Maven
      uses: stCarolas/setup-maven@d6af6abeda15e98926a57b5aa970a96bb37f97d1 # v5
      with:
        maven-version: 3.9.11
    - name: Build with Maven
      run: mvn -B clean verify

    - name: SCP artifacts to download.eclipse.org
      uses: appleboy/scp-action@ff85246acaad7bdce478db94a363cd2bf7c90345 # v1
      with:
        host: projects-storage.eclipse.org
        username: ${{ secrets.SCP_USERNAME }}
        key: ${{ secrets.SCP_KEY }}
        passphrase: ${{ secrets.SCP_PASSPHRASE }}
        source: "org.eclipse.cbi.tycho.example.updatesite/target/org.eclipse.cbi.tycho.example.updatesite-*.zip"
        target: "/home/data/httpd/download.eclipse.org/cbi/github-upload-test/"
        strip_components: 2

```
{% endraw %}

Used here: https://github.com/eclipse-cbi/eclipse-cbi-tycho-example/blob/main/.github/workflows/deploy-to-download-server.yml

Build artifacts are uploaded to https://download.eclipse.org/cbi/github-upload-test/.

## Best practices

* Regularily clean up your download directory

## Resources

* https://github.com/appleboy/scp-action
