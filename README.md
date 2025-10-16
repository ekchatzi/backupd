# Backupd

A flexible backup solution for databases and directories with support for retention policies.

## Features

- **MySQL Database Backups**: Dump, compress (gzip), and rsync to target
- **Directory Backups**: Direct rsync with customizable options
- **Flexible Credentials**: Load from `.env` files, JSON files, or environment variables
- **Placeholder System**: Reference credentials and variables anywhere in config
- **Retention Policies**: Keep daily/weekly/monthly/yearly backups automatically
- **Cron-Ready**: Silent by default, only outputs errors
- **No External Dependencies**: Uses only Python standard library

## Installation

```bash
# Make the script executable
chmod +x backupd

# Optionally, symlink to a directory in your PATH
sudo ln -s $(pwd)/backupd /usr/local/bin/backupd
```

## Configuration

Create a `backup.json` file in your project directory:

```json
[
    {
        "env": "./.env",
        "type": "mysql",
        "db_user": "{env.DB_USERNAME}",
        "db_password": "{env.DB_PASSWORD}",
        "db_host": "{env.DB_SERVER}",
        "db_database": "{env.DB_DATABASE}",
        "target": "/backups/db/{hostname}/",
        "compress": "gzip",
        "previous_versions": [
            {"interval": "1D", "count": 14},
            {"interval": "1W", "count": 8},
            {"interval": "1M", "count": 6},
            {"interval": "1Y", "count": 2}
        ]
    },
    {
        "type": "directory",
        "source": "/var/www/uploads/",
        "target": "/backups/files/",
        "options": ["--delete"]
    }
]
```

### Backup Types

#### MySQL Backup (`type: "mysql"`)

Required fields:
- `db_user`: Database username
- `db_password`: Database password
- `db_database`: Database name to dump
- `target`: Destination for backup files

Optional fields:
- `db_host`: Database host (default: `localhost`)
- `compress`: Compression method (default: `gzip`)
- `env`: Path to .env file with credentials
- `previous_versions`: Retention policy (see below)

#### Directory Backup (`type: "directory"`)

Required fields:
- `source`: Source directory path
- `target`: Destination directory path

Optional fields:
- `options`: Array of rsync options (e.g., `["--delete", "--exclude=*.tmp"]`)

### Placeholder System

Use placeholders anywhere in your configuration:

#### Environment Variables
```json
{
    "db_password": "{env.DB_PASSWORD}"
}
```

Loads from:
1. `.env` file specified in the job
2. System environment variables

#### JSON Files
```json
{
    "db_user": "{file.credentials.db.username}"
}
```

Loads from `credentials.json` and accesses nested keys with dot notation.

#### Built-in Placeholders
- `{hostname}`: System hostname
- `{datetime}`: Current timestamp (format: `YYYYMMDD_HHMMSS`)
- `{date}`: Current date (format: `YYYYMMDD`)
- `{time}`: Current time (format: `HHMMSS`)
- `{year}`, `{month}`, `{day}`: Individual date components

Example:
```json
{
    "target": "/backups/{hostname}/{database}_{datetime}.sql.gz"
}
```

### Retention Policies

Define how many backups to keep using `previous_versions`:

```json
"previous_versions": [
    {"interval": "1D", "count": 14},  // Keep 1 per day for 14 days
    {"interval": "1W", "count": 8},   // Keep 1 per week for 8 weeks
    {"interval": "1M", "count": 6},   // Keep 1 per month for 6 months
    {"interval": "1Y", "count": 2}    // Keep 1 per year for 2 years
]
```

Interval units:
- `D`: Days
- `W`: Weeks
- `M`: Months (30 days)
- `Y`: Years (365 days)

**Note**: Retention policies currently only work for local targets. Remote rsync:// retention is not yet implemented.

### Environment File Format

Create a `.env` file:

```bash
# Database credentials
DB_USERNAME=myuser
DB_PASSWORD=mypassword
DB_SERVER=localhost
DB_DATABASE=mydatabase
```

## Usage

```bash
# Run backups using ./backup.json
./backupd

# Use a custom config file
./backupd -c /etc/backup.json

# Verbose output (useful for testing)
./backupd -v

# Dry run (test without executing)
./backupd --dry-run -v
```

## Cron Setup

Add to your crontab for automated backups:

```bash
# Edit crontab
crontab -e

# Run backups every day at 2 AM
0 2 * * * /path/to/backupd -c /path/to/backup.json

# Run backups every 6 hours
0 */6 * * * /path/to/backupd -c /path/to/backup.json

# Run with verbose output to a log file
0 2 * * * /path/to/backupd -c /path/to/backup.json -v >> /var/log/backupd.log 2>&1
```

## Examples

### Example 1: Simple MySQL Backup

```json
[
    {
        "type": "mysql",
        "db_user": "root",
        "db_password": "secret",
        "db_database": "myapp",
        "target": "/backups/mysql/",
        "compress": "gzip"
    }
]
```

### Example 2: Remote Rsync Backup

```json
[
    {
        "type": "directory",
        "source": "./data/",
        "target": "rsync://u123456@backup.example.com:23/myapp/",
        "options": ["--delete"]
    }
]
```

### Example 3: Complete Setup with Credentials

**backup.json:**
```json
[
    {
        "env": "./.env",
        "type": "mysql",
        "db_user": "{env.DB_USER}",
        "db_password": "{env.DB_PASS}",
        "db_host": "{env.DB_HOST}",
        "db_database": "production",
        "target": "rsync://backup@storage.com:23/db/{hostname}/",
        "compress": "gzip",
        "previous_versions": [
            {"interval": "1D", "count": 7},
            {"interval": "1W", "count": 4}
        ]
    }
]
```

**.env:**
```bash
DB_USER=admin
DB_PASS=secure_password
DB_HOST=localhost
```

## Requirements

- Python 3.6+
- `mysqldump` command (for MySQL backups)
- `rsync` command
- `gzip` command (for compression)

All dependencies are system commands - no Python packages needed!

## Exit Codes

- `0`: All backups completed successfully
- `1`: One or more backups failed

## Troubleshooting

### "Configuration file not found"
Make sure `backup.json` exists in the current directory, or use `-c` to specify the path.

### "Environment variable not found"
Check that your `.env` file exists and contains the required variables, or that they're set in the system environment.

### "mysqldump failed"
- Verify MySQL credentials are correct
- Check that `mysqldump` is installed and in PATH
- Ensure the database exists

### "rsync failed"
- Verify source and target paths are correct
- For remote targets, check SSH access and credentials
- Ensure sufficient permissions and disk space

## License

Public Domain / Do Whatever You Want With It
