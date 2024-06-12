# HOWTO: Mount M365 GCCH OneDrive Business on Linux using `rclone`

## Goal
The goal is to mount a M365 GCCH OneDrive Business account, which almost always requires Multi-Factor Authentication (MFA), on a Linux machine using rclone. This setup needs to ensure secure access and optionally handle MFA, while complying with GCCH requirements `"us"` region. We configure both the server side (Azure) and the client side (rclone). We ensure proper handling of OAuth2 tokens using GCCH endpoints.

## Prerequisites
- **Administrative Access**: You need administrative access to the Azure portal. You may need administrative access to the Linux machine for `rclone` and/or the `expect` script.
- **rclone**: Ensure `rclone` is installed on your Linux machine.
- **Command-Line Skills**: Basic familiarity with Linux command-line operations.

## Installation and Setup

### Install or Update `rclone`

Note that the version available in `apt` was two years outdated in current repos when this was written.

1. **Install or Update rclone**:
    - Download and install the latest version of `rclone` from [rclone.org](https://rclone.org/):
    ```bash
    curl https://rclone.org/install.sh | sudo bash
    ```

2. **Verify Installation**:
    - Check the installed version to ensure it is up to date:
    ```bash
    rclone version
    ```

3. **Backup Configuration (if updating)**:
    - Backup your existing `rclone` configuration file before updating:
    ```bash
    cp ~/.config/rclone/rclone.conf ~/.config/rclone/rclone.conf.backup
    ```

4. **Restore Configuration (if updating)**:
    - Restore your configuration file after the update:
    ```bash
    mv ~/.config/rclone/rclone.conf.backup ~/.config/rclone/rclone.conf
    ```

5. **Stop Running Instances (if updating)**:
    - Stop any running `rclone` mounts or daemons before updating:
    ```bash
    killall rclone
    ```

6. **Verify Post-Update**:
    - Verify the updated version to ensure the installation was successful:
    ```bash
    rclone version
    ```

## Server Setup (Azure)

### Log in to Azure Portal
1. Access the [Azure Portal](https://portal.azure.us/) and log in with your administrative account.

### Register a New Application
2. Navigate to **Azure Active Directory** -> **App registrations** -> **New registration**.
   - Provide a name for the application (e.g., "RcloneOneDriveGCCH").
   - Set the "Supported account types" to "Accounts in this organizational directory only".
   - Set the "Redirect URI" type to "Public client/native (mobile & desktop)".
   - Enter `http://localhost:53682/` as the Redirect URI. This is the rclone default.
   - Click **Register**.

### Create Client ID and Secret
3. After registration, go to **Certificates & secrets** -> **New client secret**.
   - Add a description (e.g., "RcloneSecret") and set an expiration period.
   - Click **Add** and copy the generated secret value. You will need this later.
   - Note the "Application (client) ID" on the application's overview page.

### Set API Permissions
4. Navigate to **API permissions** -> **Add a permission** -> **Microsoft Graph**.
   - Select **Delegated permissions**.
   - Add the following permissions:
     - `Files.Read`
     - `Files.Read.All`
     - `Files.ReadWrite`
     - `Files.ReadWrite.All`
     - `User.Read`
   - Click **Add permissions**.
   - Ensure to **Grant admin consent** for the added permissions.


## Client Configuration (rclone)

### rclone Configuration with `expect` Script

You may have to go through `rclone config` a few times to get it right. Here's an `expect` script to speed up the easier steps of the basic setup and then stop at the "Use auto config" step, where you'll need to complete the authorization manually.

```expect
#!/usr/bin/expect -f

# Set the variables. rclone seems to have no awareness of tenant ids
set remote_name "yourRemoteName"
set client_id "YourClientID"
set client_secret "YourClientSecret"
set tenant_id "YourTenantID.onmicrosoft.us"
# Perform string substitution for URLs
set auth_url "https://login.microsoftonline.us/$tenant_id/oauth2/v2.0/authorize"
set token_url "https://login.microsoftonline.us/$tenant_id/oauth2/v2.0/token"
# Environment variables to ensure rclone uses the correct URLs
set env(RCLONE_ONEDRIVE_AUTH_URL) $auth_url
set env(RCLONE_ONEDRIVE_TOKEN_URL) $token_url

set timeout -1
#spawn rclone config
spawn -noecho env RCLONE_ONEDRIVE_AUTH_URL=$env(RCLONE_ONEDRIVE_AUTH_URL) RCLONE_ONEDRIVE_TOKEN_URL=$env(RCLONE_ONEDRIVE_TOKEN_URL) rclone config

expect "e/n/d/r/c/s/q>" { send "n\r" }
expect "name>" { send "$remote_name\r" }
expect "Storage>" { send "onedrive\r" }
expect "client_id>" { send "$client_id\r" }
expect "client_secret>" { send "$client_secret\r" }
expect "region>" { send "us\r" }
expect "y/n>" { send "y\r" } ;#advanced config
expect "OAuth Access Token as a JSON blob." { send "\r" }
expect "auth_url>" { send "$env(RCLONE_ONEDRIVE_AUTH_URL)\r" }
expect "token_url>" { send "$env(RCLONE_ONEDRIVE_TOKEN_URL)\r" }
expect "chunk_size>" { send "\r" }
expect "drive_id>" { send "\r" }
expect "drive_type>" { send "business\r" }
expect "root_folder_id>" { send "\r" }
expect "access_scopes>" { send "\r" }
expect "expose_onenote_files>" { send "\r" }
expect "server_side_across_configs>" { send "\r" }
expect "list_chunk>" { send "\r" }
expect "no_versions>" { send "\r" }
expect "link_scope>" { send "\r" }
expect "link_type>" { send "\r" }
expect "link_password>" { send "\r" }
expect "hash_type>" { send "\r" }
expect "av_override>" { send "\r" }
expect "delta>" { send "\r" } 
expect "metadata_permissions>" { send "\r" }  
expect "encoding>" { send "\r" }
expect "description>" { send "\r" } 
expect "y/n>" { send "n\r" } ;#lucky people can change this for autoconfig
expect "y/n>" { send "n\r" } ;#lucky people can change this for browser

puts "\n                     ***STOP HERE***"
puts "                  ***RETURN TO THE GUIDE AND USE THIS LINE***"
puts "rclone authorize \"onedrive\" --auth-no-open-browser \"$client_id\" \"$client_secret\" -vv"
puts "                  *** IGNORE WHAT IT SAYS BELOW ***\n"

interact ;# return control to the user
```
### Complete rclone Configuration

#### Manual Authorization
Avoid letting rclone auto-fetch the token. Instead, manually run the authorization server/listener yourself, which the expect script will also provide for you:
```bash
rclone authorize "onedrive" --auth-no-open-browser "client_id" "client_secret" -vv
```
The listener will also give you the browser URL to visit. Do that.

You will get a big chunk of output in your terminal. Look for:
```plaintext
Paste the following into your remote machine --->
<---End paste
```
Everything in between those lines goes into the value for 'token' in your rclone config that is still waiting for you from before you manually ran the listener `rclone authorize` command.

Proceed through the final steps of the `rclone` config wizard.


### Mount Your Drive

1. **Create Mount Point Directory**:
```   bash
   mkdir -p ~/rclone/gcch
```
2. **Mount OneDrive**:
   - Use the following command to mount OneDrive:
     ```bash
     rclone mount onedrive_gcch: ~/rclone/gcch --vfs-cache-mode full --daemon
     ```

### Troubleshooting

If you encounter issues, use the following curl command to manually request a token:

```bash
curl -X POST https://login.microsoftonline.us/tenant_id/oauth2/v2.0/token \
-d client_id=client_id \
-d client_secret=client_secret \
-d redirect_uri=http://localhost:53682/ \
-d grant_type=authorization_code \
-d code=AUTH_CODE
```

## References

### Example config
```ini
[onedrive_gcch]
type = onedrive
client_id = YourClientID
client_secret = YourClientSecret
region = us
auth_url = https://login.microsoftonline.us/YourTenantID.onmicrosoft.us/oauth2/v2.0/authorize
token_url = https://login.microsoftonline.us/YourTenantID.onmicrosoft.us/oauth2/v2.0/token
drive_type = business
delta = true
drive_id = YourDriveID
token = {"access_token":"xxx","token_type":"Bearer","refresh_token":"xxx","expiry":"2020-01-11T17:01:11.400110811-01:00"}
```ini

### Links

- [rclone OneDrive Page](https://rclone.org/onedrive/): Note the Business section and some arguments for the US region.
- [Remote Setup](https://rclone.org/remote_setup/): For more help and alternate methods.
- [Microsoft 365 GCC High Endpoints](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-u-s-government-gcc-high-endpoints?view=o365-worldwide)
- [National Cloud Deployments](https://github.com/abraunegg/onedrive/blob/master/docs/national-cloud-deployments.md): Alternate solution to rclone.

