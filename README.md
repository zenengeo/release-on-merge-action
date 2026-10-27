# Release on Merge Action

A GitHub Action that automatically creates semantic versioned GitHub Releases when a pull request is merged **with PR release labels**.

The original motivation was to streamline the process of accepting a Dependabot or Renovate dependency update PR and subsequently bumping the release version of the project to pick up the change. 

## Features

- **Semantic Versioning**: Automatically calculates the next requested semantic version (`patch`, `minor`, or `major`) based on the repository's latest release.
- **Automated Releases**: Creates a new GitHub Release with automatically generated release notes when a labeled PR is merged. The merged commit is used as the release-tag to ensure a concurrent merge does not interfere with the release.
- **Prefix Preservation**: Retains version prefix formatting (e.g., `v1.2.3` vs `1.2.3`) matching your repository's existing release naming style.
- **Step Outputs**: Exports the calculated/created version string (`release-version`) for downstream workflow jobs.

---

## How It Works

> [!NOTE]
> The action will quietly do nothing if a `release/*` label is not present on the PR.

1. The action triggers when a pull request is closed and merged.
2. It inspects the labels on the pull request for a label starting with `release/` (`release/patch`, `release/minor`, or `release/major`).
3. If a valid release label is present, it fetches the repository's latest release.
4. It parses the semantic version and increments it according to the label type.
5. Unless `dry-run` is enabled, it creates a new GitHub Release with the incremented tag/name and automatically generated release notes.
6. Sets the `release-version` output.

```
Previous                               
release                                
======                         Released
v1.5.3        PR Label         version 
 │ │ │        =============    ======  
 │ │ └──────► release/patch -> v1.5.4  
 │ └────────► release/minor -> v1.6.0  
 └──────────► release/major -> v2.0.0  
```

---

## Prerequisites

1. **Initial Release**: The repository must have at least one existing release with a semantic version name (e.g., `v1.0.0` or `1.0.0`) so the action can determine the baseline version.
2. **Release Labels**: Create the following labels in your repository:
   - `release/patch` (for bug fixes and patch updates)
   - `release/minor` (for new backward-compatible features)
   - `release/major` (for breaking changes)

---

## Usage

### Basic Example

Add a workflow file (e.g. `.github/workflows/release-on-merge.yml`) to your repository:

```yaml
name: Release on merge

on:
   pull_request_target:
      types: [closed]

jobs:
   release:
      permissions:
         contents: write
      # Optional since release-on-merge-action will also narrow on the event state
      if: github.event.pull_request.merged == true
      runs-on: ubuntu-latest
      steps:
         - name: Run Release on Merge Action
           id: release-on-merge
           uses: itzg/release-on-merge-action@v1.1.3
```

> [!IMPORTANT]
> The `pull_request_target` event is used to trigger workflows from pull requests targeting the base branch. This event runs in the context of the base repository, which means it can access secrets and environment variables from the base repository. However, it does not have access to secrets and environment variables from the pull request repository.

### Triggering Workflows on Release Creation (Using a PAT)

Events triggered by the default `GITHUB_TOKEN` do not trigger new workflow runs. If you have separate workflows triggered by release creation (such as package publishing or deployment workflows), use a Personal Access Token (PAT) with `repo` scope:

```yaml
- name: Create Release
  uses: itzg/release-on-merge-action@v1.1.3
  with:
    token: ${{ secrets.GH_RELEASE_TOKEN }}
```

### Full example

```yaml
name: Release on merge

on:
   pull_request_target:
      types: [closed]

jobs:
   release:
      # Optional since release-on-merge-action will also narrow on the event state
      if: github.event.pull_request.merged == true
      runs-on: ubuntu-latest
      steps:
         - name: Run Release on Merge Action
           id: release-on-merge
           uses: itzg/release-on-merge-action@v1.1.3
           with:
              # Be sure to allocate and assign a Personal Access Token with write access to the repository.
              # Required when the creation of the release will need to trigger another Github workflow.
              token: ${{ secrets.GH_RELEASE_TOKEN }}

         - name: Display Release Output
           if: steps.release-on-merge.outputs.release-version != ''
           run: |
              echo "Created release version: ${{ steps.release-on-merge.outputs.release-version }}"
```

---

## Inputs

| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `token` | GitHub token used for release operations. Pass a PAT if you need release creation to trigger downstream workflows. | No | `${{ github.token }}` |
| `dry-run` | When set to `true`, calculates the next version and sets outputs without creating a release or tag. | No | `'false'` |

---

## Outputs

| Output | Description |
| --- | --- |
| `release-version` | The semantic version of the release that was created (or calculated during a dry run). |

---

## Release Labels Reference

| Label | SemVer Increment | Example (from `v1.2.3`) |
| --- | --- | --- |
| `release/patch` | Patch (`0.0.X`) | `v1.2.4` |
| `release/minor` | Minor (`0.X.0`) | `v1.3.0` |
| `release/major` | Major (`X.0.0`) | `v2.0.0` |

> **Note:** If a pull request has no `release/*` label, or has an unrecognized label, the action finishes without creating a release.

---

## Development

### Setup & Testing

```bash
# Install dependencies
npm install

# Run unit tests
npm test

# Run tests in watch mode
npm run test:watch

# Type check
npm run typecheck

# Build action bundle
npm run build
```

### Testing with `act`

You can test local GitHub Actions workflows using [act](https://github.com/nektos/act):

```bash
# Test with a patch release event
npm run test:act:patch

# Test with a minor release event
npm run test:act:minor

# Test with a major release event
npm run test:act:major

# Test with an unmerged PR event
npm run test:act
```

## Creating labels

```shell
 gh label create 'release/major' -c '#e99695' -d 'On merge, creates a new release with a major version bump.' -R OWNER/REPO
 gh label create 'release/minor' -c '#c5def5' -d 'On merge, creates a new release with a minor version bump.' -R OWNER/REPO
 gh label create 'release/patch' -c '#d4c5f9' -d 'On merge, creates a new release with a patch version bump.' -R OWNER/REPO
```

## Actions policy

This action ideally executes with the `pull_request_target` event and is safe to do so, since it does not perform a checkout of the code.

Github requires that an Actions Policy be configured that explicitly allows that event usage. In <https://github.com/OWNER/REPO/settings/actions/rules>, import that following:

```json
{
  "name": "Allow release-on-merge to use pull_request_target",
  "target": "actions",
  "source_type": "Repository",
  "enforcement": "active",
  "conditions": {
    "workflow_path": {
      "exclude": [],
      "include": [
        ".github/workflows/release-on-merge.yml"
      ]
    }
  },
  "rules": [
    {
      "type": "restrict_action_events",
      "parameters": {
        "allowed_events": [
          "pull_request_target"
        ]
      }
    }
  ],
  "bypass_actors": []
}
```
