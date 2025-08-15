# Installation Guide

This guide provides comprehensive installation instructions for the essential components of the Clinic platform: MySQL, OpenEMR, and Whisper.

## Table of Contents

- [Prerequisites](#prerequisites)
- [MySQL Installation](#mysql-installation)
- [OpenEMR Installation](#openemr-installation)
- [Whisper Installation](#whisper-installation)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before installing any components, ensure your system meets the following requirements:

- **Operating System**: Ubuntu 20.04+ / CentOS 8+ / macOS 10.15+ / Windows 10+
- **RAM**: Minimum 4GB, Recommended 8GB+
- **Storage**: At least 20GB free space
- **Network**: Internet connection for downloading packages
- **User Privileges**: Administrator/sudo access

## MySQL Installation

MySQL is the primary database management system for the clinic platform.

### Version Requirements

- **Recommended**: MySQL 8.0.x
- **Minimum**: MySQL 5.7.x

### Ubuntu/Debian Installation

```bash
# Update package repository
sudo apt update

# Install MySQL Server
sudo apt install mysql-server mysql-client

# Secure MySQL installation
sudo mysql_secure_installation
```

### CentOS/RHEL Installation

```bash
# Install MySQL repository
sudo dnf install https://dev.mysql.com/get/mysql80-community-release-el8-1.noarch.rpm

# Install MySQL Server
sudo dnf install mysql-community-server

# Start and enable MySQL service
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Get temporary password
sudo grep 'temporary password' /var/log/mysqld.log

# Secure MySQL installation
sudo mysql_secure_installation
```

### macOS Installation

```bash
# Install using Homebrew
brew install mysql

# Start MySQL service
brew services start mysql

# Secure MySQL installation
mysql_secure_installation
```

### Windows Installation

1. Download MySQL Installer from [MySQL Official Website](https://dev.mysql.com/downloads/installer/)
2. Run the installer as Administrator
3. Choose "Developer Default" setup type
4. Follow the configuration wizard
5. Set root password and create user accounts

### MySQL Configuration

1. **Login to MySQL**:
   ```bash
   mysql -u root -p
   ```

2. **Create clinic database**:
   ```sql
   CREATE DATABASE clinic_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **Create clinic user**:
   ```sql
   CREATE USER 'clinic_user'@'localhost' IDENTIFIED BY 'secure_password';
   GRANT ALL PRIVILEGES ON clinic_db.* TO 'clinic_user'@'localhost';
   FLUSH PRIVILEGES;
   ```

4. **Verify installation**:
   ```bash
   mysql -u clinic_user -p clinic_db
   ```

### MySQL Troubleshooting

**Issue**: Can't connect to MySQL server
- **Solution**: Check if MySQL service is running: `sudo systemctl status mysql`
- **Alternative**: Restart MySQL service: `sudo systemctl restart mysql`

**Issue**: Access denied for user
- **Solution**: Reset password using: `sudo mysql -u root -p`
- **Check**: User exists and has proper privileges

**Issue**: Port 3306 already in use
- **Solution**: Check for other MySQL instances: `sudo netstat -tlnp | grep :3306`
- **Alternative**: Change MySQL port in `/etc/mysql/mysql.conf.d/mysqld.cnf`

## OpenEMR Installation

OpenEMR is an open-source electronic health records (EHR) and medical practice management application.

### Prerequisites for OpenEMR

- **PHP**: Version 7.4+ (Recommended: PHP 8.1)
- **Web Server**: Apache 2.4+ or Nginx 1.18+
- **MySQL**: Already installed from previous section
- **SSL Certificate**: For production environments

### PHP Installation

#### Ubuntu/Debian

```bash
# Install PHP and required extensions
sudo apt install php php-mysql php-cli php-common php-curl php-gd php-json php-mbstring php-xml php-zip php-soap php-intl php-ldap php-fileinfo php-simplexml php-xmlwriter php-xsl

# Install Apache web server
sudo apt install apache2 libapache2-mod-php

# Enable necessary Apache modules
sudo a2enmod rewrite ssl
sudo systemctl restart apache2
```

#### CentOS/RHEL

```bash
# Install EPEL repository
sudo dnf install epel-release

# Install PHP and extensions
sudo dnf install php php-mysqlnd php-gd php-xml php-mbstring php-json php-curl php-zip php-soap php-intl php-ldap

# Install Apache
sudo dnf install httpd

# Start and enable Apache
sudo systemctl start httpd
sudo systemctl enable httpd
```

### OpenEMR Installation Steps

1. **Download OpenEMR**:
   ```bash
   cd /var/www/html
   sudo wget https://github.com/openemr/openemr/releases/download/v7_0_0/openemr-7.0.0.tar.gz
   sudo tar -xzf openemr-7.0.0.tar.gz
   sudo mv openemr-7.0.0 openemr
   ```

2. **Set proper permissions**:
   ```bash
   sudo chown -R www-data:www-data /var/www/html/openemr
   sudo chmod -R 755 /var/www/html/openemr
   ```

3. **Configure Apache Virtual Host**:
   ```bash
   sudo nano /etc/apache2/sites-available/openemr.conf
   ```

   Add the following configuration:
   ```apache
   <VirtualHost *:80>
       ServerName your-domain.com
       DocumentRoot /var/www/html/openemr
       
       <Directory /var/www/html/openemr>
           AllowOverride All
           Require all granted
       </Directory>
       
       ErrorLog ${APACHE_LOG_DIR}/openemr_error.log
       CustomLog ${APACHE_LOG_DIR}/openemr_access.log combined
   </VirtualHost>
   ```

4. **Enable the site**:
   ```bash
   sudo a2ensite openemr.conf
   sudo systemctl reload apache2
   ```

5. **Complete web-based setup**:
   - Navigate to `http://your-domain.com/openemr` in your browser
   - Follow the setup wizard
   - Use the MySQL database credentials created earlier
   - Database name: `clinic_db`
   - Database user: `clinic_user`

### OpenEMR Configuration

1. **Database Setup**:
   - Host: `localhost`
   - Port: `3306`
   - Database: `clinic_db`
   - Username: `clinic_user`
   - Password: `secure_password`

2. **Initial User Creation**:
   - Create administrator account during setup
   - Set strong password
   - Configure user preferences

3. **Security Hardening**:
   ```bash
   # Remove setup files after installation
   sudo rm -rf /var/www/html/openemr/setup.php
   sudo rm -rf /var/www/html/openemr/sql_upgrade.php
   ```

### OpenEMR Troubleshooting

**Issue**: White screen or 500 error
- **Solution**: Check PHP error logs: `sudo tail -f /var/log/apache2/error.log`
- **Check**: PHP memory limit in `/etc/php/8.1/apache2/php.ini`

**Issue**: Database connection failed
- **Solution**: Verify MySQL credentials and connection
- **Check**: MySQL service status and user permissions

**Issue**: File permission errors
- **Solution**: Reset permissions:
  ```bash
  sudo chown -R www-data:www-data /var/www/html/openemr
  sudo chmod -R 755 /var/www/html/openemr
  ```

## Whisper Installation

Whisper is OpenAI's automatic speech recognition (ASR) system for speech-to-text conversion.

### Prerequisites for Whisper

- **Python**: Version 3.8+
- **pip**: Python package installer
- **FFmpeg**: For audio processing
- **CUDA** (Optional): For GPU acceleration

### Python and pip Installation

#### Ubuntu/Debian

```bash
# Install Python and pip
sudo apt update
sudo apt install python3 python3-pip python3-venv

# Verify installation
python3 --version
pip3 --version
```

#### CentOS/RHEL

```bash
# Install Python and pip
sudo dnf install python3 python3-pip

# Verify installation
python3 --version
pip3 --version
```

#### macOS

```bash
# Install using Homebrew
brew install python3

# Verify installation
python3 --version
pip3 --version
```

#### Windows

1. Download Python from [python.org](https://www.python.org/downloads/)
2. Run installer and check "Add Python to PATH"
3. Verify installation in Command Prompt:
   ```cmd
   python --version
   pip --version
   ```

### FFmpeg Installation

#### Ubuntu/Debian

```bash
sudo apt install ffmpeg
```

#### CentOS/RHEL

```bash
# Enable RPM Fusion repository
sudo dnf install https://download1.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm

# Install FFmpeg
sudo dnf install ffmpeg
```

#### macOS

```bash
brew install ffmpeg
```

#### Windows

1. Download FFmpeg from [ffmpeg.org](https://ffmpeg.org/download.html)
2. Extract to a folder (e.g., `C:\ffmpeg`)
3. Add `C:\ffmpeg\bin` to system PATH

### Whisper Installation Steps

1. **Create virtual environment** (recommended):
   ```bash
   python3 -m venv whisper_env
   source whisper_env/bin/activate  # On Windows: whisper_env\Scripts\activate
   ```

2. **Install Whisper**:
   ```bash
   pip install openai-whisper
   ```

3. **Install additional dependencies**:
   ```bash
   pip install torch torchvision torchaudio
   ```

4. **For GPU support** (optional):
   ```bash
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
   ```

### Whisper Usage Examples

1. **Basic transcription**:
   ```bash
   whisper audio_file.mp3 --model medium
   ```

2. **Specify output format**:
   ```bash
   whisper audio_file.wav --model large --output_format txt
   ```

3. **Multiple languages**:
   ```bash
   whisper audio_file.mp3 --model medium --language Spanish
   ```

4. **Python script example**:
   ```python
   import whisper
   
   # Load model
   model = whisper.load_model("base")
   
   # Transcribe audio
   result = model.transcribe("audio_file.mp3")
   print(result["text"])
   ```

### Whisper Model Information

| Model | Parameters | VRAM | Speed |
|-------|------------|------|-------|
| tiny  | 39 M       | ~1 GB | ~32x  |
| base  | 74 M       | ~1 GB | ~16x  |
| small | 244 M      | ~2 GB | ~6x   |
| medium| 769 M      | ~5 GB | ~2x   |
| large | 1550 M     | ~10 GB| 1x    |

### Whisper Troubleshooting

**Issue**: ModuleNotFoundError for torch
- **Solution**: Install PyTorch: `pip install torch torchvision torchaudio`

**Issue**: FFmpeg not found
- **Solution**: Verify FFmpeg installation: `ffmpeg -version`
- **Alternative**: Add FFmpeg to system PATH

**Issue**: CUDA out of memory
- **Solution**: Use smaller model or reduce batch size
- **Alternative**: Use CPU-only mode: `whisper audio.mp3 --device cpu`

**Issue**: Audio format not supported
- **Solution**: Convert to supported format using FFmpeg:
  ```bash
  ffmpeg -i input.m4a output.wav
  ```

## Troubleshooting

### General Issues

**Issue**: Permission denied errors
- **Solution**: Check user permissions and use `sudo` when necessary
- **Alternative**: Verify file ownership and permissions

**Issue**: Port conflicts
- **Solution**: Check for running services: `sudo netstat -tlnp`
- **Alternative**: Change default ports in configuration files

**Issue**: Service won't start
- **Solution**: Check service logs: `sudo journalctl -u service_name`
- **Alternative**: Verify configuration files for syntax errors

### Performance Optimization

1. **MySQL Optimization**:
   ```sql
   # Increase buffer pool size
   innodb_buffer_pool_size = 1G
   
   # Optimize query cache
   query_cache_size = 256M
   query_cache_type = 1
   ```

2. **Apache Optimization**:
   ```apache
   # Enable compression
   LoadModule deflate_module modules/mod_deflate.so
   
   # Set keep-alive
   KeepAlive On
   MaxKeepAliveRequests 100
   KeepAliveTimeout 15
   ```

3. **PHP Optimization**:
   ```ini
   # Increase memory limit
   memory_limit = 512M
   
   # Optimize opcache
   opcache.enable = 1
   opcache.memory_consumption = 128
   ```

### Security Considerations

1. **Database Security**:
   - Use strong passwords
   - Limit user privileges
   - Enable SSL connections
   - Regular security updates

2. **Web Server Security**:
   - Keep software updated
   - Use SSL certificates
   - Configure firewalls
   - Regular backups

3. **Application Security**:
   - Change default passwords
   - Enable audit logging
   - Regular security patches
   - User access controls

## Next Steps

After completing the installation:

1. **Test all components** to ensure they work together
2. **Configure backups** for database and application files
3. **Set up monitoring** for system performance and uptime
4. **Review security settings** and apply hardening measures
5. **Train users** on the system functionality

For additional support, consult the official documentation:
- [MySQL Documentation](https://dev.mysql.com/doc/)
- [OpenEMR Documentation](https://www.open-emr.org/wiki/index.php/OpenEMR_Documentation)
- [Whisper Documentation](https://github.com/openai/whisper)