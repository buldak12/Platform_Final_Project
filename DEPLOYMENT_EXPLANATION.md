# Deployment Configuration Explanation

## Part 1: Dockerfile Setup (1-2 minutes)

### Overview
Our Dockerfile uses a **multi-stage build** approach to create an optimized Docker image for a Symfony PHP application with Nginx.

### Stage 1: Builder Stage
```dockerfile
FROM php:8.3-fpm as builder

WORKDIR /app
```
- Uses PHP 8.3 FPM as the base image
- Creates `/app` as the working directory
- This stage prepares dependencies and application code

```dockerfile
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    curl \
    nodejs \
    npm \
    && docker-php-ext-install pdo pdo_mysql \
    && rm -rf /var/lib/apt/lists/*
```
- Installs required system packages (git, unzip, curl, Node.js, npm)
- Installs PHP extensions: `pdo` and `pdo_mysql` for database connectivity
- Cleans up apt cache to reduce image size

```dockerfile
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
```
- Downloads and installs Composer (PHP dependency manager)
- Makes it globally available

```dockerfile
COPY composer.json composer.lock ./
RUN composer install --no-interaction --no-scripts --optimize-autoloader
```
- Copies dependency files
- Installs PHP dependencies without running post-install scripts yet

```dockerfile
COPY . .
RUN composer install --no-interaction --optimize-autoloader --no-ansi || true
RUN php bin/console importmap:install --no-interaction
RUN php bin/console cache:warmup --env=prod --no-debug || true
```
- Copies entire application code
- Runs post-install scripts and builds optimized autoloader
- Installs asset imports and warms up the production cache

### Stage 2: Runtime Stage
```dockerfile
FROM php:8.3-fpm as runtime

WORKDIR /app

RUN apt-get update && apt-get install -y \
    nginx \
    curl \
    && docker-php-ext-install pdo pdo_mysql \
    && rm -rf /var/lib/apt/lists/*
```
- Creates final runtime image with PHP 8.3 FPM
- Installs only what's needed: Nginx web server and curl for health checks
- Installs same PHP extensions

```dockerfile
COPY --from=builder /app /app
```
- Copies entire prepared application from builder stage
- This keeps the final image smaller (builder stage is discarded)

```dockerfile
RUN mkdir -p /app/var && \
    chown -R www-data:www-data /app && \
    chmod -R 755 /app && \
    chmod -R 775 /app/var
```
- Creates var directory for cache and logs
- Sets ownership to `www-data` (web server user)
- Sets proper permissions (775 for var allows writing)

```dockerfile
COPY nginx-main.conf /etc/nginx/nginx.conf
COPY nginx.conf /etc/nginx/conf.d/symfony.conf
```
- Copies Nginx configuration files

```dockerfile
EXPOSE 8080

CMD ["sh", "-c", "php bin/console doctrine:migrations:migrate --no-interaction || true && php-fpm -D && nginx -g 'daemon off;'"]
```
- Exposes port 8080 (Railway's dynamic port)
- **Startup command:**
  1. Runs database migrations (with || true to ignore errors)
  2. Starts PHP-FPM as daemon (`-D` flag)
  3. Starts Nginx in foreground mode (keeps container alive)

---

## Part 2: Nginx Configuration (1-2 minutes)

### Overview
Nginx acts as a reverse proxy, serving static files directly and forwarding PHP requests to PHP-FPM.

```nginx
server {
    listen 8080 default_server;
    root /app/public;
    index index.php;
```
- Listens on port 8080
- Sets document root to `/app/public` (Symfony's public directory)
- Default index file is `index.php`

```nginx
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log warn;
```
- Logs all requests and errors
- Helps with debugging

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
```
- Security headers to protect against common web attacks

```nginx
location ~ ^/assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
    try_files $uri =404;
}
```
- **Static Assets:** Serves assets (CSS, JS, images) with 1-year cache
- Browser won't re-download unchanged files

```nginx
location ~ /\.well-known {
    allow all;
}

location ~ /\. {
    deny all;
}

location ~ ~$ {
    deny all;
}
```
- Security rules:
  - Allow `.well-known` (for SSL certificates)
  - Block all hidden files (starting with .)
  - Block backup files (ending with ~)

```nginx
location / {
    try_files $uri $uri/ /index.php$is_args$args;
}
```
- **URL Rewriting:** Symfony routing
- Tries to serve the file/directory
- If not found, forwards to `index.php` (Symfony's front controller)
- Preserves query strings

```nginx
location ~ ^/index\.php {
    fastcgi_pass localhost:9000;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    include fastcgi_params;
    
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
```
- **PHP Handler:** Forwards requests to PHP-FPM on port 9000
- `fastcgi_split_path_info`: Properly handles PATH_INFO for routing
- `fastcgi_params`: Includes standard FastCGI parameters
- Sets correct script filename for PHP execution

---

## How They Work Together

```
User Request
    ↓
[Nginx - Port 8080]
    ↓
  Static file?  → Serve directly (CSS, JS, images)
    ↓
  PHP file?     → Forward to PHP-FPM (port 9000)
    ↓
[PHP-FPM]
    ↓
[Symfony Application]
    ↓
  Response returned to Nginx
    ↓
  Response sent to user
```

## Key Points

✅ **Multi-stage build** keeps final image small  
✅ **Nginx + PHP-FPM** better performance than built-in servers  
✅ **Proper permissions** prevent permission errors  
✅ **Auto-migrations** run on container startup  
✅ **Security headers** protect the application  
✅ **URL rewriting** enables clean Symfony routing  
✅ **Port 8080** compatible with Railway's dynamic port assignment  

