# Publishing to repo.eclipse.org using GitHub Actions

This guide outlines how to publish Maven-based Eclipse projects directly to [repo.eclipse.org](https://repo.eclipse.org) using GitHub Actions.

## Prerequisites

* Account and namespace already claimed. [Open a HelpDesk Issue](https://gitlab.eclipse.org/eclipsefdn/helpdesk/-/issues/new)

* Publishing credentials stored as GitHub secrets

This is the list of secrets needed to deploy to repo.eclipse.org declared in GitHub organization as secrets.

* `REPO_TOKEN_USERNAME`
* `REPO_TOKEN_PASSWORD`

Otterdog configuration: e.g: https://github.com/eclipse-cbi/.eclipsefdn/blob/main/otterdog/eclipse-cbi.jsonnet

```js
secrets+: [
    orgs.newOrgSecret('REPO_TOKEN_USERNAME') {
      value: "vault:<project_id>/repo.eclipse.org/token-username",
    },
    orgs.newOrgSecret('REPO_TOKEN_PASSWORD') {
      value: "vault:<project_id>/repo.eclipse.org/token-password",
    },
  ],
```

## GitHub Actions Workflow Example

Example of snapshot deployment:

{% raw %}
```yaml
name: Publish snapshot packages to repo.eclipse.org
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
      - name: Set up repo.eclipse.org Repository
        uses: actions/setup-java@de7274f081f381c8f8158605e0321c36c376e2e6 # v6.0.1
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: repo.eclipse.org
          server-username: MAVEN_USERNAME
          server-password: MAVEN_PASSWORD
      - name: Publish package
        run: mvn -P release --batch-mode deploy
        env:
          MAVEN_USERNAME: ${{ secrets.REPO_TOKEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.REPO_TOKEN_PASSWORD }}
```
{% endraw %}

## Best Practices

- Sign all artifacts (even on snapshots)
