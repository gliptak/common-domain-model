# Quick Start Guide - GitHub Actions Workflow

This guide helps you get started with the CDM GitHub Actions build and deployment workflow.

## Prerequisites

Before the workflow can run successfully, ensure:

1. **GitHub Repository Secrets are Configured**
   - See [SECRETS.md](SECRETS.md) for the complete list
   - At minimum, you need the GPG and Maven Central credentials

2. **Permissions**
   - Ensure GitHub Actions is enabled for the repository
   - Verify workflow permissions in Settings > Actions > General

## Running the Workflow

### Automatic Triggers

The workflow runs automatically on:

1. **Push to master/main branch**
   ```bash
   git push origin master
   ```
   - Builds a SNAPSHOT version
   - Does NOT deploy to Maven Central
   - Does NOT tag the repository

2. **Push a tag**
   ```bash
   git tag v5.0.0
   git push origin v5.0.0
   ```
   - Builds a release version (uses the tag name)
   - DEPLOYS to Maven Central (requires all secrets)
   - Tags the repository after successful build

3. **Pull Request**
   ```bash
   # Create a PR via GitHub UI or CLI
   gh pr create --title "My changes"
   ```
   - Builds and tests the code
   - Does NOT deploy or tag

### Manual Trigger

You can manually trigger the workflow from the GitHub UI:

1. Go to **Actions** tab in the repository
2. Select **CDM Build and Deploy** workflow
3. Click **Run workflow** button
4. Select the branch to run from
5. Click **Run workflow**

## Monitoring the Workflow

### Viewing Workflow Runs

1. Go to the **Actions** tab in the repository
2. Click on a specific workflow run to see details
3. Click on individual jobs to see logs

### Understanding Job Status

- ✅ **Green check** - Job completed successfully
- ❌ **Red X** - Job failed
- 🟡 **Yellow dot** - Job in progress
- ⚪ **Gray circle** - Job queued or skipped

### Parallel Jobs

These jobs run in parallel after the main build:
- `build-daml` - DAML build
- `build-scala` - Scala build  
- `build-csharp8` - C# 8 build
- `build-csharp9` - C# 9 build
- `build-python` - Python build

They run simultaneously for faster feedback.

## Build Artifacts

### Download Build Artifacts

1. Go to a completed workflow run
2. Scroll to the **Artifacts** section at the bottom
3. Click to download:
   - `maven-artifacts` - Main Maven build outputs
   - `daml-artifacts` - DAML build outputs
   - `scala-artifacts` - Scala build outputs
   - `csharp8-artifacts` - C# 8 build outputs
   - `csharp9-artifacts` - C# 9 build outputs
   - `python-artifacts` - Python build outputs

### Artifact Retention

- Artifacts are kept for **7 days** by default
- Download artifacts before they expire if needed

## Troubleshooting

### Build Fails on Main Maven Build

1. Check the `build` job logs
2. Look for Maven error messages
3. Common issues:
   - Dependency resolution failures
   - Compilation errors
   - Test failures

### Build Fails on Parallel Jobs

1. Check specific job logs (e.g., `build-daml`)
2. These jobs have `continue-on-error: true`
3. They won't fail the entire pipeline but should be investigated

### Deployment Fails

1. Check deploy job logs (e.g., `deploy-daml`)
2. Verify secrets are configured correctly
3. Common issues:
   - GPG key import failure → Check `GPG_PRIVATE_KEY` format
   - Maven Central authentication → Check `CI_DEPLOY_USERNAME` and `CI_DEPLOY_PASSWORD`
   - Signing failure → Check `GPG_PASSPHRASE`

### Tag Push Fails

1. Check the `tag-release` job logs
2. Common issues:
   - Tag already exists
   - Insufficient permissions on `REGNOSYS_OPS_TOKEN`
   - Token expired

## Release Process

### Creating a Release Build

1. **Ensure main branch is stable**
   ```bash
   git checkout master
   git pull
   ```

2. **Create and push a version tag**
   ```bash
   git tag v5.0.0
   git push origin v5.0.0
   ```

3. **Monitor the workflow**
   - Go to Actions tab
   - Watch the build progress
   - Verify all parallel builds succeed
   - Confirm deployments complete

4. **Verify deployment**
   - Check Maven Central for published artifacts
   - Verify tag was created in repository

### Version Naming Convention

- **Release**: `v5.0.0` (any tag)
- **Snapshot**: `0.0.0.main-SNAPSHOT` (branch builds)

The workflow automatically uses the tag name for releases.

## Common Commands

### Check Workflow Status (CLI)

```bash
# List recent workflow runs
gh run list --workflow=build-deploy.yml

# View specific run details
gh run view <run-id>

# Watch a running workflow
gh run watch <run-id>

# View logs for a specific job
gh run view <run-id> --log --job=<job-id>
```

### Re-run Failed Workflow

```bash
# Re-run all failed jobs
gh run rerun <run-id> --failed

# Re-run entire workflow
gh run rerun <run-id>
```

### Download Artifacts (CLI)

```bash
# Download all artifacts from a run
gh run download <run-id>

# Download specific artifact
gh run download <run-id> -n maven-artifacts
```

## Performance Tips

### Build Times

Expected build times:
- **Setup**: ~30 seconds
- **Main Maven Build**: ~10-15 minutes
- **Parallel Builds**: ~5-10 minutes (run simultaneously)
- **Deployments**: ~2-3 minutes per artifact (only on releases)

Total time:
- **Snapshot build**: ~15-20 minutes
- **Release build**: ~30-40 minutes (includes all deployments)

### Caching

The workflow uses Maven caching to speed up builds:
- First build: Downloads all dependencies (~10-15 min)
- Subsequent builds: Uses cached dependencies (~5-10 min)

Cache key: `${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}`

## Best Practices

1. **Test on Feature Branch First**
   - Create a PR to test changes
   - Verify build succeeds before merging

2. **Monitor Parallel Builds**
   - Check all parallel jobs for warnings
   - Fix issues even if they don't fail the pipeline

3. **Verify Secrets Regularly**
   - Rotate credentials periodically
   - Check for expiration on tokens
   - Update GPG keys if needed

4. **Review Deployment Logs**
   - Always verify artifacts were published correctly
   - Check for warnings in deployment logs

5. **Keep Workflow Updated**
   - Review workflow for improvements
   - Update action versions regularly
   - Monitor GitHub Actions changelog

## Need Help?

- **Workflow issues**: Check [MIGRATION.md](MIGRATION.md) for detailed technical info
- **Secret configuration**: See [SECRETS.md](SECRETS.md)
- **CDM questions**: Contact [cdm-maintainers@lists.finos.org](mailto:cdm-maintainers@lists.finos.org)
- **GitHub Actions docs**: https://docs.github.com/en/actions

## Next Steps

1. Configure all required secrets
2. Test with a feature branch PR
3. Create a test tag to verify release build
4. Monitor first few production releases
5. Update branch protection rules if needed
