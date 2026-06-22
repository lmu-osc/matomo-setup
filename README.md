# OSC Matomo Analytics Setup

This repository contains the Docker Compose configuration for deploying [Matomo](https://matomo.org/) (formerly Piwik), the open-source web analytics platform, at the [Open Science Center](https://www.osc.uni-muenchen.de/) of LMU Munich.

## Overview

The stack consists of three Docker services defined in `docker-compose.yml`:

| Service  | Container Name | Image              | Purpose                                      |
|----------|---------------|--------------------|----------------------------------------------|
| `db`     | `matomo-db`   | `mariadb:latest`   | MariaDB database for Matomo                  |
| `matomo` | `matomo-app`  | `matomo:latest`    | Main Matomo application (web interface)      |
| `cron`   | `matomo-cron` | `matomo:latest`    | Periodic report archiving via cron            |

### Services Explained

#### `db` — MariaDB Database
- Runs the latest MariaDB image with a `64MB` max packet size.
- Creates a `matomo` database automatically via the `MARIADB_DATABASE` environment variable.
- Persists data to `./db` on the host via a bind mount.
- Credentials are sourced from environment variables (see [Environment Variables](#environment-variables)).

#### `matomo` — Matomo Application
- The main Matomo instance, listening on `127.0.0.1:8080` (mapped to port 80 inside the container).
- Configured via environment variables to connect to the `db` service.
- Persists its configuration, plugins, and uploaded files to `./matomo` on the host.
- Includes the `extra_hosts` entry needed for SMTP/email delivery (see [Email & SMTP Setup](#email--smtp-setup)).
- Depends on the `db` service starting first.

#### `cron` — Archiving Cron
- Runs an infinite loop that executes `core:archive` every 5 minutes (300 seconds).
- This processes visit data into pre-computed reports, keeping the UI fast.
- **Important:** In Matomo's _System > General Settings > Archiving Settings_, "Archive reports when viewed from the browser" should be set to **"No"** when using cron-based archiving. This is set manually in the browser after initial setup.

## Set-Up on Our Server

The Matomo service is already set-up on our server so you should not have to repeat these steps. However, included here are the generally steps you would need to take to do this:

1. **Clone the repository:**
   ```bash
   git clone git@github.com:lmu-osc/matomo-setup.git
   cd matomo-setup
   ```

2. **Create the environment file:**
   ```bash
   cp .env-example .env
   ```
   Edit `.env` and fill in secure values for the database credentials.

3. **Start the services:**
   ```bash
   docker compose up -d
   ```

4. **Run the Matomo web installer:**
   Most instructions would indicate you should open the app at e.g. `http:localhost:8080` (or whatever port number is used--8080 is the port used by Docker for the matomo-app here), and then follow the Matomo setup wizard instructions using the database credentials from your `.env` file. However, our server is not configured to have a browser/UI. Instead, assuming the Nginx setup for the site is still properly configured, you can simply go to our analytics web address to initiate the set-up.

> **Note:** This setup is based on the [official Matomo Docker installation guide](https://matomo.org/faq/how-to-install/install-matomo-with-docker/), adapted to our naming conventions and existing volume structure.

## Environment Variables

The following variables are expected in the `.env` file (see `.env-example`):

| Variable                | Description                            |
|-------------------------|----------------------------------------|
| `MARIADB_USER`          | Matomo database user                   |
| `MARIADB_PASSWORD`      | Password for the Matomo database user  |
| `MARIADB_ROOT_PASSWORD` | Password for the MariaDB root user     |

## Troubleshooting & Setup Notes

### File Permissions

After the first `docker compose up -d`, you may encounter errors like:

> The directory "/var/www/html/tmp/cache/tracker/" does not exist and could not be created.

or

> The Matomo configuration file (config/config.ini.php) is not writable, some of your changes might not be saved.

To resolve these, stop the stack and set the appropriate permissions:

```bash
docker compose down
chmod -R 777 ./matomo/
```

This ensures the `matomo` user inside the container can write to the `tmp/`, `config/`, and other directories. Then restart the stack:

```bash
docker compose up -d
```

### LogViewer Plugin — Log File Configuration

If you install the [LogViewer](https://plugins.matomo.org/LogViewer) plugin from the Matomo Marketplace, you may see:

> Specified path to log file does not exist: /tmp/logs/matomo.log

This means Matomo is not yet configured to write logs to a file (by default it logs to stderr, which goes to `docker logs`). In our case, the fix was to:

1. **Create the log file:**

   ```bash
   # cd to the root, i.e. where the Docker Compose file lives
   touch ./matomo/tmp/logs/matomo.log
   ```

2. **Configure Matomo to write to the log file** by adding the following to `./matomo/config/config.ini.php`:

   ```ini
   # https://matomo.org/faq/troubleshooting/faq_115/
   [log]
   log_writers[] = file
   log_level = "INFO"

   [Debug]
   enable_sql_profiler = 1
   ```

   This tells Matomo to use the file writer in addition to (or instead of) the default stderr output.

### Nginx Configuration — Private Directory Blocking

The Nginx config on the server (`/etc/nginx/sites-available/www.analytics.osc.lmu.de`) includes a rule that denies access to Matomo's private directories at the reverse proxy level:

```nginx
# Deny access to Matomo's private directories
location ~ ^/(config|tmp|core|lang|misc|node_modules) {
    deny all;
    return 404;
}
```

This ensures these sensitive paths (configuration files, cache, etc.) are never served to the public, even if Matomo itself would serve them. If you see warnings in _System → Diagnostics → Required Private Directories_ about connection failures to these paths, this is expected — the check tries to reach the public URL from inside the Docker container, which won't work in our setup. The directories are properly blocked by Nginx.

### Email & SMTP Setup

Sending emails from Matomo (e.g., password resets, scheduled reports) requires SMTP configuration. Our setup uses **Postfix** on the Docker host as an SMTP relay. That is, the matomo-app Docker container relays messages out of the container, and then uses the server's Postfix configuration to send emails.

#### 1. Configure Postfix on the Host

The Matomo container runs on a Docker internal network. You need to allow the container's IP range in Postfix:

```bash
# Find the container's IP address
docker inspect matomo-app --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# Add the Docker bridge networks to Postfix's trusted networks
sudo postconf -e "mynetworks = 127.0.0.0/8, 172.17.0.0/16, 172.18.0.0/16"
sudo systemctl restart postfix
```

Note that the IP address will *likely* be in the included CIDR ranges already, but it's good to check that the new IP address after a restart is still encompassed by the included networks trusted by Postfix.

> **Note:** The Docker network IP range may change across environments or if containers are recreated. A more robust approach would be to configure Postfix as a full SMTP relay with proper authentication, or to use an external SMTP service.

#### 2. Enable `host.docker.internal` in Docker Compose

The `matomo` and `cron` services include:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

This makes the Docker host reachable from within the container at the hostname `host.docker.internal`.

#### 3. Configure Matomo to Use SMTP

In Matomo's web interface, go to _System > Mail Server Settings_ and set the SMTP server address to `host.docker.internal`.

Alternatively, edit `./matomo/config/config.ini.php` directly:

```ini
[General]
noreply_email_address = "root+lmpp10e-osc@srv.mwn.de"
noreply_email_name = "Matomo Analytics OSC"
<OTHER CONTENT UNDER GENERAL>

[mail]
transport = "smtp"
port = "25"
host = "host.docker.internal"
encryption = "none"
```

#### 4. Test Email Delivery

After editing the config file (if that was needed), test the email relay works:

```bash
docker compose down
docker compose up -d
docker exec matomo-app php /var/www/html/console core:test-email CONTACT@lmu.de
```

> You should only ever be adding LMU domain email addresses to our Matomo so you only need to test against such emails. 

### Cron Archiving

The `cron` service runs `core:archive` every 5 minutes. You can verify it's working by checking the logs:

```bash
docker logs matomo-cron
# or use the -f flag to follow the logs live
# docker logs matomo-cron -f
```

On first inspection, the Matomo System Check may report that the last archive was long ago (e.g., ~40 days). This is normal if archiving was previously done manually — the cron service will pick it up on its next cycle. A full archive run typically completes in about 6 minutes.

### Updating the Repository on the Server

When pulling changes to this repo on the production server:

```bash
git pull
docker compose down
docker compose up -d
```

Make sure the server user has the correct filesystem permissions and a working SSH connection to GitHub.

## File Structure

```
.
├── .env-example          # Template for required environment variables
├── .gitignore
├── README.md             # This file
├── docker-compose.yml    # Docker Compose service definitions
├── db/                   # MariaDB data (git-ignored)
└── matomo/               # Matomo application files (git-ignored)
```