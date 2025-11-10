# Keystore Compatibility Fix

## Problem

The build fails with the following error:

```
com.android.ide.common.signing.KeytoolException: Failed to read key from store
"keystore.jks": Tag number over 30 is not supported
```

## Root Cause

This error occurs when the Android Gradle Plugin's signing tools encounter a keystore format that uses ASN.1 tags not supported by the current JDK implementation. This typically happens when:

1. A keystore was created with Java 9+ using the default PKCS12 format
2. The PKCS12 keystore uses certain extensions or advanced features
3. The Android signing tools don't fully support these newer PKCS12 features

Even though the project uses Java 21 and the CI environment is configured correctly, the keystore format itself is incompatible with the Android build tools.

## Solution

The keystore needs to be recreated with explicitly compatible settings. We provide a script that creates a new keystore using:

- **Keystore Type**: JKS (Java KeyStore) instead of PKCS12 for maximum compatibility
- **Key Algorithm**: RSA 2048-bit (industry standard for Android apps)
- **Signature Algorithm**: SHA256withRSA (secure and widely supported)
- **Validity**: 10,000 days (~27 years) to avoid expiration issues

## Steps to Fix

### 1. Run the Keystore Recreation Script

```bash
./scripts/recreate-keystore.sh
```

The script will:
- Prompt you for keystore credentials (passwords and key alias)
- Prompt you for certificate details (CN, OU, O, L, ST, C)
- Create a new compatible keystore at `app/keystore.jks`
- Optionally create a base64-encoded version for GitHub Secrets

### 2. Update GitHub Secrets

After running the script, you need to update the following GitHub repository secrets:

1. Go to your repository on GitHub
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Update or create these secrets:
   - `RELEASE_KEYSTORE_BASE64`: Content of the `keystore.base64` file
   - `KEYSTORE_PASSWORD`: The keystore password you entered
   - `KEY_ALIAS`: The key alias you entered (default: `release`)
   - `KEY_PASSWORD`: The key password you entered

### 3. Clean Up Sensitive Files

After uploading to GitHub Secrets:

```bash
# Remove the base64 file
rm keystore.base64

# Remove the local keystore (it's in GitHub Secrets now)
rm app/keystore.jks

# The keystore is recreated during CI from the secret
```

### 4. Test the Build

Trigger a new build by:
- Pushing a new tag (for releases)
- Using workflow_dispatch in the GitHub Actions UI
- Pushing to your branch

The build should now complete successfully without the "Tag number over 30" error.

## Important Notes

### For Existing Apps

⚠️ **WARNING**: If you already have an app published on Google Play or other stores, you **MUST** use the same signing key. Creating a new keystore will generate a different key, and you won't be able to update your existing app.

**If you need to recover or reuse an existing key:**
- If you have a backup of the old keystore and remember the password, use that instead
- If you lost the keystore, you cannot update the published app and will need to publish as a new app
- Store your new keystore and credentials securely to avoid this issue in the future

### For New Apps

If this is a new app that hasn't been published yet, you can safely create a new keystore with the script.

### Security Best Practices

1. **Store credentials securely**: Use a password manager for the keystore password, key password, and key alias
2. **Backup the keystore**: Keep an encrypted backup of your keystore in a secure location
3. **Never commit to Git**: The keystore should never be committed to version control
4. **Use strong passwords**: The keystore password should be strong and unique

## Alternative: Manual Keystore Creation

If you prefer to create the keystore manually:

```bash
keytool -genkeypair \
    -keystore app/keystore.jks \
    -alias release \
    -keyalg RSA \
    -keysize 2048 \
    -sigalg SHA256withRSA \
    -validity 10000 \
    -storetype JKS \
    -dname "CN=Your Name, OU=Development, O=Your Org, L=City, ST=State, C=US"
```

Then convert to base64:

```bash
base64 -w 0 app/keystore.jks > keystore.base64
```

## Verification

After recreating the keystore, you can verify it with:

```bash
keytool -list -v -keystore app/keystore.jks
```

This will show:
- Keystore type: JKS
- Key algorithm: RSA
- Signature algorithm: SHA256withRSA
- Certificate validity period

## References

- [Android App Signing Documentation](https://developer.android.com/studio/publish/app-signing)
- [Keytool Documentation](https://docs.oracle.com/en/java/javase/21/docs/specs/man/keytool.html)
- [Issue Discussion](https://github.com/gradle/gradle/issues/17847)
