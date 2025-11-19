# nginx-site

A Bash script to manage nginx-based websites with dedicated system users, automatic SSL certificates, and optional systemd services for applications.

## Features

- **Simple Setup**: Create static or application sites with a single command
- **Git-Based Deployments**: All projects are deployed from Git repositories
- **Automatic SSL**: Integrated certbot for automatic HTTPS setup
- **Systemd Integration**: Application sites get automatic systemd service management
- **Isolated Users**: Each project runs under its own system user
- **Clean Structure**: Organized directory layout for configs, logs, and site files

## Prerequisites

Before using nginx-site, ensure these tools are installed on your system:

- `nginx` - Web server
- `git` - Version control for deployments
- `certbot` - SSL certificate management

**Installation example (Debian/Ubuntu):**
```bash
sudo apt update
sudo apt install nginx git certbot python3-certbot-nginx
```

## Installation

1. Clone or download the `nginx-site` script:
   ```bash
   git clone https://github.com/yourusername/nginx-site.git
   cd nginx-site
   ```

2. Make it executable and move to your PATH:
   ```bash
   chmod +x nginx-site
   sudo mv nginx-site /usr/local/bin/
   ```

3. **Configure your main domain** (REQUIRED):
   ```bash
   sudo nginx-site setup yourdomain.com
   ```

   This creates `/etc/nginx-site.conf` with your domain configuration. All sites will be created as subdomains of this domain.

### About Subdomains

**All sites are automatically created as subdomains of your main domain.**

The subdomain is derived from your Git repository name:
- Repository: `github.com/user/myblog` → Creates `myblog.example.com`
- Repository: `github.com/user/api` → Creates `api.example.com`
- Repository: `github.com/user/my-awesome-app` → Creates `my-awesome-app.example.com`

This approach:
- **Automatic**: No need to specify subdomains manually
- **Consistent**: Repository name = subdomain name = project name
- **Organized**: All sites under one domain
- **Simple SSL**: Automatic certificate management per subdomain

**To reconfigure your domain:**
```bash
sudo nginx-site setup newdomain.com
```

**Configuration file location:** `/etc/nginx-site.conf`

## Architecture

Each project follows this structure:

```
/home/{project}/
├── config/
│   ├── nginx.conf          # Nginx configuration (symlinked to sites-enabled)
│   └── .env                # Environment variables (apps only)
├── site/                   # Git repository cloned here
└── logs/                   # Nginx access and error logs
```

**Principles:**
- One project = one system user = one home directory
- Git-based deployments only
- Simple, opinionable templates

## Usage

### Dry Run Mode

All commands that modify the system support a `--dry-run` flag to preview changes without executing them:

```bash
# Preview what would happen when creating a site
# Subdomain is derived from repo name: blog.yourdomain.com
sudo nginx-site create static --repo https://github.com/user/blog.git --dry-run

# Preview service start
sudo nginx-site start myproject --dry-run

# Preview removal
sudo nginx-site remove myproject --dry-run
```

In dry-run mode:
- Commands are echoed with `[DRY-RUN]` prefix instead of being executed
- File contents (nginx configs, systemd services, .env files) are displayed
- No actual changes are made to the system
- Interactive confirmations are skipped

This is useful for:
- Verifying what will be created before committing
- Understanding what files and commands are involved
- Testing configurations
- Documentation and learning

### Create a Static Site

```bash
# Creates myblog.yourdomain.com (subdomain derived from repo name 'myblog')
nginx-site create static --repo https://github.com/user/myblog.git
```

This creates:
- System user based on repository name (`myblog`)
- Subdomain automatically: `myblog.yourdomain.com`
- Directory structure at `/home/myblog/`
- Nginx configuration serving static files
- SSL certificate via certbot for the subdomain

**After creation:**
1. Ensure your static files are in `/home/myblog/site`
2. Run `nginx-site start myblog`

### Create an Application Site

```bash
# Creates myapi.yourdomain.com (subdomain derived from repo name 'myapi')
nginx-site create app \
  --repo https://github.com/user/myapi.git \
  --start "npm start" \
  --port 3000
```

**Optional:** Omit `--port` to auto-detect the first available port in range 3000-4000.

This creates everything from static sites, plus:
- Systemd service file
- Nginx reverse proxy configuration
- `.env` file for environment variables

**After creation:**
1. Install application dependencies: `cd /home/{project}/site && npm install`
2. Edit environment variables: `/home/{project}/config/.env`
3. Run `nginx-site start {project}`

### Manage Projects

```bash
# Start a project (enables and starts services, reloads nginx)
nginx-site start {project}

# Stop a project (stops the application service)
nginx-site stop {project}

# Restart a project (restarts the application service)
nginx-site restart {project}

# Check project status
nginx-site status {project}

# List all managed projects
nginx-site list

# Show configuration file paths
nginx-site config {project}

# Remove a project completely
nginx-site remove {project}
```

## Examples

### Static HTML Site

```bash
# Create a simple static site at myblog.yourdomain.com
# (subdomain automatically derived from repo name 'myblog')
sudo nginx-site create static --repo https://github.com/user/myblog.git

# Start the site
sudo nginx-site start myblog
```

### Node.js Application

```bash
# Create a Node.js API at myapi.yourdomain.com
# (subdomain automatically derived from repo name 'myapi')
sudo nginx-site create app --repo https://github.com/user/myapi.git --start "node index.js"

# Install dependencies
cd /home/myapi/site
sudo -u myapi npm install

# Configure environment
sudo nano /home/myapi/config/.env

# Start the service
sudo nginx-site start myapi
```

### Python Flask Application

```bash
# Create a Flask app at myflaskapp.yourdomain.com
# (subdomain automatically derived from repo name 'myflaskapp')
sudo nginx-site create app \
  --repo https://github.com/user/myflaskapp.git \
  --start "/home/myflaskapp/site/venv/bin/python app.py" \
  --port 5000

# Setup Python environment (as the project user)
cd /home/myflaskapp/site
sudo -u myflaskapp python3 -m venv venv
sudo -u myflaskapp venv/bin/pip install -r requirements.txt

# Start the service
sudo nginx-site start myflaskapp
```

## Configuration Files

### Nginx Configurations

Nginx configurations are stored in `/home/{project}/config/nginx.conf` and symlinked to `/etc/nginx/sites-enabled/{project}`.

**Static Site Template:**
- Serves files from `/home/{project}/site`
- Default index: `index.html`
- Supports clean URLs (`/about` serves `about.html`)

**Application Site Template:**
- Reverse proxy to `http://127.0.0.1:{port}`
- Proper proxy headers for IP forwarding
- 100MB max upload size
- 300s timeout for long requests

After certbot runs, SSL configuration is automatically added.

### Systemd Services

Application sites get a systemd service at `/etc/systemd/system/{project}.service`:

```ini
[Unit]
Description={project} application service
After=network.target

[Service]
ExecStart={your-command}
WorkingDirectory=/home/{project}/site
EnvironmentFile=/home/{project}/config/.env
User={project}
Group={project}
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Environment Variables

Applications can use environment variables via `/home/{project}/config/.env`:

```bash
# Example .env file
PORT=3000
NODE_ENV=production
DATABASE_URL=postgresql://localhost/mydb
API_KEY=your-secret-key
```

## Deployment Workflow

1. **Initial Setup:**
   ```bash
   sudo nginx-site create app --repo {url} --start "{command}"
   # Subdomain is automatically derived from repository name
   ```

2. **Install Dependencies:**
   ```bash
   cd /home/{project}/site
   sudo -u {project} npm install  # or pip install, etc.
   ```

3. **Configure Environment:**
   ```bash
   sudo nano /home/{project}/config/.env
   ```

4. **Start Services:**
   ```bash
   sudo nginx-site start {project}
   ```

5. **Updates (redeploy):**
   ```bash
   cd /home/{project}/site
   sudo -u {project} git pull
   sudo -u {project} npm install  # if dependencies changed
   sudo nginx-site restart {project}
   ```

## Logs

View logs using standard Linux tools:

```bash
# Nginx access logs
tail -f /home/{project}/logs/access.log

# Nginx error logs
tail -f /home/{project}/logs/error.log

# Application logs (systemd journal)
sudo journalctl -u {project}.service -f
```

## Limitations & Design Decisions

### What nginx-site Handles:
- Infrastructure setup (users, directories, configs)
- Service management (systemd, nginx)
- SSL certificates (via certbot)

### What nginx-site Does NOT Handle:
- Application dependencies (npm install, pip install, etc.)
- Build processes (npm run build, etc.)
- Dependency validation (checking if nginx/git/certbot are installed)
- Input validation (port conflicts, existing users, domain conflicts)
- Database setup
- Backup/restore functionality

### User Responsibilities:
- Installing system dependencies (nginx, git, certbot)
- Running application-specific setup after creation
- Ensuring unique ports, domains, and project names
- Managing application builds and dependencies
- Database and external service configuration

## Troubleshooting

### Service won't start

```bash
# Check service status
sudo systemctl status {project}.service

# View detailed logs
sudo journalctl -u {project}.service -n 50

# Check if port is available
sudo ss -tlnp | grep :{port}
```

### Nginx configuration errors

```bash
# Test nginx config
sudo nginx -t

# View nginx error log
sudo tail -f /var/log/nginx/error.log

# Check project-specific error log
sudo tail -f /home/{project}/logs/error.log
```

### SSL certificate issues

```bash
# Manually run certbot
sudo certbot --nginx -d {domain}

# Check certificate status
sudo certbot certificates

# Renew certificates
sudo certbot renew
```

### Permission issues

```bash
# Ensure correct ownership
sudo chown -R {project}:{project} /home/{project}

# Check file permissions
ls -la /home/{project}
```

## Security Considerations

- Each project runs under its own system user (privilege separation)
- `.env` files are created with mode 600 (owner read/write only)
- System users are created with `-r` flag (system user, no login shell by default)
- All application processes run as non-root users

## Contributing

This is a simple, opinionated tool. When contributing:

- Follow KISS and DRY principles
- Maintain pure Bash (no additional dependencies)
- Keep the code self-documenting
- Don't over-engineer solutions

## License

MIT License - Use freely, modify as needed.

## Future Enhancements (Out of Scope)

These features are intentionally not included to keep the tool simple:

- Database setup and management
- Backup/restore functionality
- Migration from existing configurations
- SSL certificate revocation on remove
- Multiple domains per project
- Custom nginx includes or complex configuration
- Application dependency management
- Build process automation
