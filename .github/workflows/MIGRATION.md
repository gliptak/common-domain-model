# Codefresh to GitHub Actions Conversion

This document outlines the conversion of the `codefresh.yml` build pipeline to GitHub Actions.

## Overview

The CDM build pipeline has been migrated from Codefresh to GitHub Actions while maintaining:
- ✅ Parallel execution of build tasks
- ✅ Multi-language support (DAML, Scala, C#, Python, TypeScript, Go, Kotlin)
- ✅ Conditional deployment for releases
- ✅ Automated repository tagging
- ✅ GPG signing of artifacts
- ✅ Maven Central deployment

## File Structure

```
.github/
  workflows/
    build-deploy.yml    # Main workflow (replaces codefresh.yml)
    SECRETS.md          # Documentation for required secrets
    cve-scanning.yml    # Existing security scanning
    license-scanning.yml # Existing license scanning
    pmd-code-scan.yml   # Existing code quality
    windows-build.yml   # Existing Windows build
```

## Key Mappings

### Codefresh Stages → GitHub Actions Jobs

| Codefresh Stage | GitHub Actions Job | Notes |
|----------------|-------------------|-------|
| `setup` | `setup` | Determines release version and build profiles |
| `build` (Build step) | `build` | Main Maven build and install |
| `build` (BuildParallelTasks) | `build-daml`, `build-scala`, `build-csharp8`, `build-csharp9`, `build-python` | Parallel execution maintained |
| `build` (Deploy steps) | `deploy-daml`, `deploy-scala`, `deploy-typescript`, etc. | Only run on release (tag) builds |
| `finalise` (TagRepo) | `tag-release` | Only run on successful release builds |
| `finalise` (FailDeployPipeline) | `check-status` | Checks all job statuses |

### Environment Variables

| Codefresh Variable | GitHub Actions Equivalent | Notes |
|-------------------|--------------------------|-------|
| `${{CF_REPO_OWNER}}` | `github.repository_owner` | Built-in context |
| `${{CF_REPO_NAME}}` | `github.event.repository.name` | Built-in context |
| `${{CF_REVISION}}` | `github.sha` | Built-in context |
| `${{CF_BRANCH_TAG_NORMALIZED}}` | `github.ref_name` (normalized) | Computed in setup job |
| `${{TAG_REPO}}` | `github.ref_type == 'tag'` | Conditional check |
| `${{TAG_NAME}}` | `github.ref_name` | For tag builds |
| `${{CF_VOLUME_PATH}}` | GitHub workspace or cache | Uses actions/cache for Maven |
| `cf_export` | `echo "var=value" >> $GITHUB_OUTPUT` | Job outputs |

### Secrets Mapping

All secrets remain the same and need to be configured in GitHub repository settings:

| Secret Name | Purpose |
|------------|---------|
| `CI_DEPLOY_USERNAME` | Maven Central username |
| `CI_DEPLOY_PASSWORD` | Maven Central password |
| `GPG_KEYNAME` | GPG key identifier |
| `GPG_PASSPHRASE` | GPG key passphrase |
| `GPG_PRIVATE_KEY` | GPG private key |
| `REGNOSYS_OPS` | GitHub username for tagging |
| `REGNOSYS_OPS_TOKEN` | GitHub PAT for pushing tags |

## Workflow Triggers

The GitHub Actions workflow triggers on:

```yaml
on:
  push:
    branches:
      - master
      - main
    tags:
      - '*'
  pull_request:
    branches:
      - master
      - main
  workflow_dispatch:  # Manual trigger
```

This provides more flexibility than Codefresh, including:
- Manual workflow runs via `workflow_dispatch`
- Automatic builds on pull requests
- Tag-based release builds

## Parallel Execution

Codefresh used a `parallel` step type:

```yaml
BuildParallelTasks:
  type: parallel
  steps:
    BuildDaml: ...
    BuildScala: ...
```

GitHub Actions achieves this through job dependencies:

```yaml
build-daml:
  needs: [setup, build]
  
build-scala:
  needs: [setup, build]
  
# Both jobs run in parallel once their dependencies complete
```

## Artifact Management

**Codefresh**: Used shared volumes (`CF_VOLUME_PATH`) to share files between steps.

**GitHub Actions**: Uses `actions/upload-artifact` and `actions/download-artifact`:

```yaml
# After main build
- uses: actions/upload-artifact@v4
  with:
    name: maven-artifacts
    path: rosetta-source/target/

# In parallel jobs
- uses: actions/download-artifact@v4
  with:
    name: maven-artifacts
```

## Docker Images

**Codefresh**: Specified docker images per step:

```yaml
BuildDaml:
  image: digitalasset/daml-sdk:1.3.0
```

**GitHub Actions**: Uses `container` property or setup actions:

```yaml
build-daml:
  steps:
    - uses: docker://digitalasset/daml-sdk:1.3.0
      
# Or for standard tools:
build-csharp8:
  steps:
    - uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '3.1.x'
```

## Error Handling

**Codefresh**: Used `fail_fast: false` and conditional failure steps.

**GitHub Actions**: Uses `continue-on-error: true` and job status checks:

```yaml
build-daml:
  continue-on-error: true
  
check-status:
  needs: [build, build-daml, ...]
  if: always()
  steps:
    - run: |
        if [[ "${{ needs.build.result }}" == "failure" ]]; then
          exit 1
        fi
```

## Conditional Execution

**Codefresh**: Used `when.condition` blocks:

```yaml
when:
  condition:
    all:
      isRelease: "${{TAG_REPO}}"
```

**GitHub Actions**: Uses `if` conditionals:

```yaml
deploy-daml:
  if: needs.setup.outputs.is-release == 'true'
```

## Benefits of GitHub Actions

1. **Native Integration**: Direct integration with GitHub features (PR checks, status badges, etc.)
2. **Cost**: Free for public repositories
3. **Marketplace**: Access to thousands of pre-built actions
4. **Visibility**: Workflow runs visible directly in the repository UI
5. **Caching**: Built-in caching for dependencies (Maven, npm, etc.)
6. **Matrix Builds**: Easy to test across multiple versions/platforms
7. **Reusable Workflows**: Can create composite actions and reusable workflows

## Migration Checklist for Maintainers

- [ ] Configure all required secrets in repository settings
- [ ] Test workflow on a feature branch first
- [ ] Verify GPG key import works correctly
- [ ] Test snapshot build (push to branch)
- [ ] Test release build (create a test tag)
- [ ] Verify Maven Central deployment
- [ ] Update branch protection rules if needed
- [ ] Monitor first few builds for issues
- [ ] Update any external tools that depend on Codefresh status
- [ ] Consider deprecating/removing codefresh.yml once stable

## Troubleshooting

### Common Issues

1. **GPG Import Fails**
   - Verify `GPG_PRIVATE_KEY` is in ASCII-armored format
   - Check for proper line endings in the secret value

2. **Maven Deploy Fails**
   - Verify Maven Central credentials are correct
   - Check that settings.xml is properly configured
   - Ensure GPG signing works

3. **Artifact Download Fails**
   - Verify artifact names match between upload and download
   - Check that the dependent job completed successfully
   - Ensure artifacts haven't expired (default: 7 days retention)

4. **Tag Push Fails**
   - Verify `REGNOSYS_OPS_TOKEN` has `repo` permissions
   - Check that the token hasn't expired
   - Ensure tag doesn't already exist

## Future Enhancements

Potential improvements to consider:

1. **Matrix Builds**: Test across multiple Java versions
2. **Caching Optimization**: Cache Docker layers for faster builds
3. **Dependabot**: Automatic dependency updates
4. **Security Scanning**: CodeQL integration for security analysis
5. **Test Reporting**: Better test result visualization
6. **Deployment Environments**: Use GitHub Environments for deployment protection
7. **Composite Actions**: Extract common steps into reusable actions

## Support

For questions or issues with the GitHub Actions workflow:
- Check workflow run logs in GitHub Actions tab
- Review [SECRETS.md](.github/workflows/SECRETS.md) for configuration
- Contact CDM maintainers via [cdm-maintainers@lists.finos.org](mailto:cdm-maintainers@lists.finos.org)
