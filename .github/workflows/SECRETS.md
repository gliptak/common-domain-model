# GitHub Actions Secrets Configuration

This document lists all the secrets required to be configured in GitHub repository settings for the `build-deploy.yml` workflow.

## Required Secrets

### Maven Central Deployment
These secrets are required for deploying release artifacts to Maven Central (Sonatype OSSRH):

1. **`CI_DEPLOY_USERNAME`**
   - **Description**: Maven Central (OSSRH) deployment username
   - **Used in**: Maven deployment commands for publishing release artifacts to Maven Central
   - **Format**: String (username)
   - **Required for**: Release builds only

2. **`CI_DEPLOY_PASSWORD`**
   - **Description**: Maven Central (OSSRH) deployment password or token
   - **Used in**: Maven deployment commands for authentication to Maven Central
   - **Format**: String (password/token)
   - **Required for**: Release builds only

### GitHub Packages Deployment
These secrets are automatically provided by GitHub Actions for deploying snapshot artifacts:

- **`GITHUB_TOKEN`**
  - **Description**: Automatically generated token for GitHub Actions
  - **Used in**: Maven deployment commands for publishing snapshot artifacts to GitHub Packages
  - **Format**: Automatic (no configuration needed)
  - **Required for**: Snapshot builds only
  - **Note**: This is automatically provided by GitHub Actions; no manual configuration needed

### GPG Signing
These secrets are required for signing artifacts with GPG:

3. **`GPG_KEYNAME`**
   - **Description**: GPG key identifier/name used for signing artifacts
   - **Used in**: Maven GPG plugin configuration
   - **Format**: String (key ID or email associated with the key)

4. **`GPG_PASSPHRASE`**
   - **Description**: Passphrase for the GPG private key
   - **Used in**: Maven GPG plugin for unlocking the private key
   - **Format**: String (passphrase)

5. **`GPG_PRIVATE_KEY`**
   - **Description**: Complete GPG private key in ASCII-armored format
   - **Used in**: Importing the GPG key for signing artifacts
   - **Format**: Multi-line string (ASCII-armored GPG private key)
   - **How to export**: `gpg --armor --export-secret-keys YOUR_KEY_ID`

### GitHub Repository Operations
These secrets are required for tagging releases in the repository:

6. **`REGNOSYS_OPS`**
   - **Description**: GitHub username for automated operations
   - **Used in**: Git configuration for tagging releases
   - **Format**: String (GitHub username)

7. **`REGNOSYS_OPS_TOKEN`**
   - **Description**: GitHub Personal Access Token (PAT) with repository write permissions
   - **Used in**: Authenticating git operations for pushing tags
   - **Format**: String (GitHub PAT)
   - **Required Permissions**: 
     - `repo` (Full control of private repositories)
     - `contents:write` (for pushing tags)

## How to Configure Secrets

1. Navigate to your GitHub repository
2. Go to **Settings** > **Secrets and variables** > **Actions**
3. Click **New repository secret**
4. Add each secret with the exact name listed above
5. Paste the corresponding value
6. Click **Add secret**

## Security Notes

- Never commit these secrets directly in code
- Rotate secrets periodically, especially tokens and passwords
- Use GitHub's secret scanning to detect accidentally exposed secrets
- Limit access to repository settings to trusted maintainers only
- For GPG keys, ensure the private key is exported securely and stored safely

## Workflow Behavior

- **Release builds** (triggered by tags): 
  - Uses Maven Central secrets (`CI_DEPLOY_USERNAME`, `CI_DEPLOY_PASSWORD`)
  - Requires GPG secrets for artifact signing
  - Deploys to Maven Central via Sonatype plugin
  
- **Snapshot builds** (triggered by pushes to branches): 
  - Uses `GITHUB_TOKEN` (automatically provided)
  - No GPG signing required
  - Deploys to GitHub Packages
  
- **Pull requests**: 
  - Build and test only, no deployment
  - No secrets required for basic PR validation

## Testing Configuration

To test if secrets are properly configured:

1. Trigger a workflow run manually using `workflow_dispatch`
2. Check the workflow logs for authentication errors
3. For GPG issues, verify the key can be imported successfully
4. For Maven Central issues, verify credentials with a test deployment

## Support

For issues with secret configuration:
- Check GitHub Actions workflow logs for specific error messages
- Verify secret names match exactly (they are case-sensitive)
- Ensure no extra whitespace in secret values
- Contact repository maintainers for access issues
