# Sourcegraph Cloud SSO Setup - OneLogin  
  
This guide walks you through configuring SAML Single Sign-On (SSO) between your Sourcegraph Cloud instance and OneLogin.  
  
## Prerequisites

- **Sourcegraph Cloud Enterprise license** (SAML is Enterprise-only)
- **OneLogin administrator access**
- **Sourcegraph site administrator access**
- Your Sourcegraph Cloud instance URL (e.g., `<YOUR-SOURCEGRAPH-URL>`)

## Variable Legend

Throughout this guide, replace these placeholders with your actual values:
- `<YOUR-SOURCEGRAPH-URL>` = Your Sourcegraph Cloud instance URL (e.g., `https://company.sourcegraph.com`)
- `<YOUR-ONELOGIN-SUBDOMAIN>` = Your OneLogin organization subdomain (e.g., `mycompany`)
- `<ISSUER-URL>` = The Issuer URL from OneLogin's SSO tab (e.g., `https://mycompany.onelogin.com/saml/metadata/123456`)

## Part 1: Configure OneLogin SAML Application  
  
### Step 1: Create SAML Application

1.  **Navigate to OneLogin Apps**
      - Go to `https://<YOUR-ONELOGIN-SUBDOMAIN>.onelogin.com/apps/find`
2.  **Search for SAML Connector**
      - Type "saml" in the search field
      - Select "**SAML Custom Connector (Advanced)**" (SAML 2.0)
      - Click "**Save**"
3.  **Name Your Application**
      - Give it a descriptive name like "Sourcegraph"
      - Add an icon if desired

### Step 2: Configure Application Settings

#### Configuration Tab
Set the following properties:
- **Audience**: `<YOUR-SOURCEGRAPH-URL>/.auth/saml/metadata`
- **Recipient**: `<YOUR-SOURCEGRAPH-URL>/.auth/saml/acs`
- **ACS (Consumer) URL**: `<YOUR-SOURCEGRAPH-URL>/.auth/saml/acs`
- **ACS URL Validator**: `<YOUR-SOURCEGRAPH-URL>\\/\\.auth\\/saml\\/acs` (regex pattern)

> **Note**: The ACS URL Validator is a regular expression pattern that matches your ACS URL.

#### Parameters Tab
Configure the following SAML attributes:
- **Email** (NameID): Email
- **DisplayName**: First Name - Include in SAML Assertion: ✓
- **login**: AD user name - Include in SAML Assertion: ✓

### Step 3: Save and Record Metadata

1.  **Save the application** in OneLogin
2.  Navigate to the "**SSO**" tab
3.  **Record the Issuer URL** under "Issuer URL"
      - Should look like: `https://<YOUR-ONELOGIN-SUBDOMAIN>.onelogin.com/saml/metadata/123456`
      - Or: `https://app.onelogin.com/saml/metadata/123456`

## Part 2: Configure Sourcegraph  
  
### Step 1: Access Site Configuration

1. Log into Sourcegraph as a site administrator
2. Navigate to **Site admin → Configuration**
3. Edit your site configuration

### Step 2: Add SAML Provider  
  
Add the following configuration to your site config:  
  
```json
{
  // ... existing configuration
  "externalURL": "<YOUR-SOURCEGRAPH-URL>",
  "auth.providers": [
    {
      "type": "saml",
      "configID": "onelogin",
      "identityProviderMetadataURL": "<ISSUER-URL>"
    }
  ]
}
```

#### Important Configuration Notes:

  - **externalURL**: Must match exactly with no trailing slash
  - **identityProviderMetadataURL**: Use the Issuer URL from OneLogin
  - While optional, we strongly encourage including “allowSignup”:true as the last property in the oneLogin object within auth.providers. This will allow user accounts to be created for users when attempting to login to your Sourcegraph environment for the first time
  - **Only one SAML provider**: Sourcegraph supports only 1 SAML provider at a time

## Part 3: Testing and Validation  
  
### Step 1: Test Configuration

1. **Save your site configuration**
2. **Check for errors** in:
   - Site admin → Configuration (look for validation errors)
   - Server logs for SAML-related errors

### Step 2: Assign Users in OneLogin

1. Go to OneLogin **Applications → Sourcegraph**
2. Navigate to "**Users**" tab
3. **Assign users** who should have access to Sourcegraph
4. Set appropriate **role mappings** if configured

### Step 3: Test SSO Flow

1. **Open an incognito/private browser window**
2. Navigate to your Sourcegraph instance
3. Click "**Sign in with SAML**" (or similar button)
4. You should be redirected to OneLogin
5. After successful OneLogin authentication, you should be redirected back to Sourcegraph

## Validation Checklist

Use this checklist to verify each step completed successfully:

| Step | What to Check | Expected Result | Where to Verify |
|------|---------------|-----------------|-----------------|
| OneLogin App Created | SAML app exists | "Sourcegraph" app listed | OneLogin Apps dashboard |
| Configuration URLs | All URLs populated | No empty fields | OneLogin Configuration tab |
| Metadata URL | Issuer URL copied | Valid URL format | OneLogin SSO tab |
| Sourcegraph Config | JSON valid | No syntax errors | Site admin → Configuration |
| Users Assigned | Users have access | Users listed | OneLogin Users tab |
| SAML Button | Button appears | "Sign in with SAML" visible | Sourcegraph login page |
| SSO Redirect | Redirect works | OneLogin login page loads | Browser test |
| Login Success | User logged in | Dashboard accessible | Sourcegraph interface |

## Troubleshooting  
  
### Common Issues

#### 1. Metadata URL Not Working
If identityProviderMetadataURL doesn't work:  
      
```json
{
  "type": "saml",
  "configID": "onelogin",
  "identityProviderMetadata": "<?xml version=\"1.0\" encoding=\"utf-8\"?><EntityDescriptor xmlns=\"urn:oasis:names:tc:SAML:2.0:metadata\">...</EntityDescriptor>"
}
```

Download the metadata XML from OneLogin and escape it as a JSON string.

#### 2. URL Mismatch Errors
Ensure all URLs match exactly:
- OneLogin ACS URL = `<YOUR-SOURCEGRAPH-URL>/.auth/saml/acs`
- Sourcegraph externalURL = `<YOUR-SOURCEGRAPH-URL>`
- No trailing slashes
- Same protocol (http/https)

#### 3. User Attribute Mapping Issues
Check that OneLogin is sending the required attributes:
- **Email**: Used for user identification
- **DisplayName**: Used for user display name
- **login**: Used for username

#### 4. Certificate Issues
If you see certificate errors:
- Verify OneLogin certificate is valid
- Check system time synchronization
- Consider using custom SP certificates if needed

### Debug Logs  
  
Enable SAML debug logging in Sourcegraph by setting the environment variable:
```
INSECURE_SAML_LOG_TRACES=1
```

This will log all SAML requests and responses. Contact Sourcegraph Cloud Support for log analysis if needed.

Look specifically for: `Error prefetching SAML service provider metadata`  
  
## Security Considerations  
  
### Recommended Settings

1. **Force HTTPS**: Always use HTTPS for production
2. **Certificate Validation**: Ensure proper certificate validation
3. **User Provisioning**: Consider automatic user creation policies
4. **Session Management**: Configure appropriate session timeouts
5. **Role Mapping**: Set up proper role/permission mapping

### User Management  
```json
{
  "type": "saml",
  "configID": "onelogin",
  "identityProviderMetadataURL": "...",
  "allowSignup": true         // Allow new user creation via SAML
} 
```
  
## Emergency Access Recovery

⚠️ **BEFORE YOU START**: Ensure you have a builtin admin account active before enabling SAML.

If you get locked out after SAML configuration:

1. **Log in with builtin admin credentials** at `<YOUR-SOURCEGRAPH-URL>/sign-in`
2. **Remove SAML configuration**:
   - Navigate to Site admin → Configuration
   - Remove the SAML provider from `auth.providers`:
   ```json
   {
     "externalURL": "<YOUR-SOURCEGRAPH-URL>",
     "auth.providers": []
   }
   ```
   - Save configuration

3. **Contact Sourcegraph Cloud Support** if locked out: support@sourcegraph.com
   - Include your Cloud instance URL

**Cloud Safety Tips**:
- Always maintain an active builtin admin account
- Test SAML with one user before rolling out broadly  
- Assign multiple users as site admins for redundancy

## Next Steps  
  
After successful SAML setup:

1. **Configure repository permissions** if needed
2. **Set up team/organization mappings**
3. **Test with multiple users**
4. **Document the login process** for your team
5. **Plan backup authentication** (keep builtin auth as fallback)

## Resources

- [Sourcegraph SAML Documentation](https://sourcegraph.com/docs/admin/auth/saml)
- [OneLogin SAML Setup Guide](https://sourcegraph.com/docs/admin/auth/saml/one_login)
- [Sourcegraph Site Configuration Reference](https://sourcegraph.com/docs/admin/config/site_config)
- [Sourcegraph SAML Troubleshooting](https://sourcegraph.com/docs/admin/auth/saml/troubleshooting)
