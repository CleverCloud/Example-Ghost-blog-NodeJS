# Ghost 6 on Clever Cloud

Deploy [Ghost](https://ghost.org/), an open source publishing platform for websites, memberships and newsletters, on Clever Cloud with the Linux runtime, [Mise](https://mise.jdx.dev/), MySQL, Cellar S3-compatible object storage and an FS Bucket.

This example was tested with Ghost `6.60.0`. MySQL stores publications, users and settings, Cellar stores uploaded media, and the FS Bucket persists themes and routing configuration uploaded from Ghost Admin.

## Prerequisites

- A [Clever Cloud account](https://console.clever-cloud.com/)
- [Clever Tools](https://www.clever.cloud/developers/doc/cli/) configured for that account
- [Git](https://git-scm.com/downloads)
- [jq](https://jqlang.org/download/)
- [s3cmd](https://s3tools.org/s3cmd) or another S3-compatible client

## Prepare the repository

Clone this repository:

```bash
git clone https://github.com/CleverCloud/Example-Ghost-blog-NodeJS.git myGhost
cd myGhost
```

The `mise.toml` file downloads Ghost, installs its production dependencies, seeds its default content and defines the `build` and `run` tasks automatically used by the Clever Cloud Linux runtime.

## Create the application and add-ons

Create a Linux application with an alias, then create and link MySQL 8.4, Cellar and FS Bucket add-ons:

```bash
clever create -t linux -a myGhost

clever addon create mysql-addon myGhostMySQL -p xxs_sml --addon-version 8.4 -l myGhost
clever addon create cellar-addon myGhostCellar -l myGhost
clever addon create fs-bucket myGhostContent -l myGhost
```

Clever Tools targets your personal organisation by default. To use another organisation, add `--org ORGANISATION` or `-o ORGANISATION` to commands that support it.

Display the application URL. You can also add a [custom domain](https://www.clever.cloud/developers/doc/administrate/domain-names/), which requires DNS configuration:

```bash
clever domain
clever domain add your.website.tld
```

Mount the linked FS Bucket on the persistent Ghost content directory:

```bash
FS_BUCKET_HOST="$(clever env -F json | jq -er 'first(.fromAddons[] | select(.addonName == "myGhostContent") | .env[] | select(.name == "BUCKET_HOST") | .value)')"
clever env set CC_FS_BUCKET "/content:${FS_BUCKET_HOST}"
unset FS_BUCKET_HOST
```

The name-based lookup requires a unique add-on name. If several linked add-ons use the same name, use the add-on ID returned by `clever addon create` instead:

```bash
FS_BUCKET_HOST="$(clever addon env ADDON_ID -F json | jq -er '.BUCKET_HOST')"
```

## Configure Cellar

Set the exact HTTPS origin returned by `clever domain`, without a trailing slash, and choose a globally unique bucket name containing only lowercase letters, numbers, dots and hyphens:

```bash
export GHOST_URL="https://your-ghost-domain.example.com"
export GHOST_BUCKET="your-unique-ghost-bucket"
```

Load the Cellar credentials injected by the linked add-on and create the bucket:

```bash
source <(clever env -F shell | grep '^export CELLAR_ADDON_')

s3cmd --access_key="$CELLAR_ADDON_KEY_ID" \
  --secret_key="$CELLAR_ADDON_KEY_SECRET" \
  --host="$CELLAR_ADDON_HOST" \
  --host-bucket="$CELLAR_ADDON_HOST" \
  --ssl mb "s3://$GHOST_BUCKET"
```

Ghost returns direct URLs for uploaded assets, so visitors need public read access to bucket objects. Apply a policy that allows reads without allowing public writes or bucket listing:

```bash
jq --null-input --arg bucket "$GHOST_BUCKET" '{
  Version: "2012-10-17",
  Statement: [{
    Sid: "PublicRead",
    Effect: "Allow",
    Principal: "*",
    Action: "s3:GetObject",
    Resource: "arn:aws:s3:::\($bucket)/*"
  }]
}' > ghost-bucket-policy.json

s3cmd --access_key="$CELLAR_ADDON_KEY_ID" \
  --secret_key="$CELLAR_ADDON_KEY_SECRET" \
  --host="$CELLAR_ADDON_HOST" \
  --host-bucket="$CELLAR_ADDON_HOST" \
  --ssl setpolicy ghost-bucket-policy.json "s3://$GHOST_BUCKET"

rm ghost-bucket-policy.json
unset CELLAR_ADDON_KEY_ID CELLAR_ADDON_KEY_SECRET CELLAR_ADDON_HOST
```

## Configure Ghost

Set the tested Ghost and Node.js versions, the public URL and the Cellar bucket name:

```bash
clever env set GHOST_VERSION 6.60.0
clever env set CC_NODE_VERSION 22.23.1

clever env set GHOST_BUCKET "$GHOST_BUCKET"
clever env set NODE_ENV production
clever env set url "$GHOST_URL"
```

The linked add-ons provide their credentials to the application. The `mise.toml` file maps them to Ghost's nested configuration and configures its listening address, port and persistent storage.

### Configure email

Ghost requires transactional email for invitations, password resets, member sign-ins and Admin verification on new devices. Configure the SMTP service of your choice and replace every example value with your provider's settings:

```bash
clever env set mail__transport SMTP
clever env set mail__from "Ghost <ghost@your.website.tld>"
clever env set mail__options__host smtp.example.com
clever env set mail__options__port 465
clever env set mail__options__secure true
clever env set mail__options__auth__user SMTP_USERNAME
clever env set mail__options__auth__pass A_STRONG_SMTP_PASSWORD
```

See the [Ghost mail configuration](https://docs.ghost.org/config/#mail) for provider-specific options. Transactional email works with standard SMTP services, but Ghost's built-in bulk newsletter delivery [requires Mailgun](https://docs.ghost.org/newsletters/#bulk-email-configuration).

If SMTP is not available during the initial setup, temporarily disable Admin verification on new devices:

```bash
clever env set security__staffDeviceVerification false
```

Before using Ghost in production, configure SMTP, re-enable device verification and restart the application:

```bash
clever env set security__staffDeviceVerification true
clever restart
```

## Deploy Ghost

Deploy the application:

```bash
clever deploy
```

Open the application and append `/ghost` to its URL to create the publication owner account:

```bash
clever open
```

The public publication is available at `$GHOST_URL`, while Admin is available at `$GHOST_URL/ghost`.

## Update Ghost

Export your content from Ghost Admin and create a recent [MySQL backup](https://www.clever.cloud/developers/doc/cli/addons/#database-backups) before updating. Check the target release's [`engines.node` requirement](https://github.com/TryGhost/Ghost/blob/v6.60.0/ghost/core/package.json), update both version variables when necessary, then rebuild without the deployment cache. For example:

```bash
clever env set GHOST_VERSION 6.60.0
clever env set CC_NODE_VERSION 22.23.1
clever restart --without-cache
```

The rebuild downloads the selected release and installs its production dependencies. Ghost applies pending MySQL migrations when the new version starts, while MySQL, Cellar and the FS Bucket preserve the publication data.

## Contributing

Contributions that improve this deployment example are welcome. Open an issue or submit a pull request with your proposed changes.
