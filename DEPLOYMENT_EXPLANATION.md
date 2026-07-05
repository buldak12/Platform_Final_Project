# Deployment Configuration Explanation

## Part 1: Dockerfile Setup (1-2 minutes)

### Overview
I use a multi-stage build approach to create an optimized Docker image for a Symfony PHP application with Nginx. This technique allows me to keep the final image small by discarding the build stage dependencies after compilation.

### Stage 1: Builder Stage
The first stage starts with PHP 8.3 FPM as the base image and creates `/app` as the working directory. This stage is responsible for preparing all the dependencies and application code that I need.

First, I install the required system packages including git, unzip, curl, Node.js, and npm. I also install PHP extensions for `pdo` and `pdo_mysql` to enable database connectivity. After installation, I clean up the apt cache to reduce the image size. This step ensures I have all the tools needed to build my application.

Next, I download and install Composer, which is the PHP dependency manager. This is installed globally so it can be accessed throughout the build process. I then copy the composer.json and composer.lock files into the container and run composer install to install all PHP dependencies. During this initial install, I skip scripts since I don't have the full application code yet.

After that, I copy the entire application code into the container. I run composer install again to execute any post-install scripts that need the full application. I also run the importmap:install command to set up asset imports, and I warm up the production cache to improve startup performance. The `|| true` at the end of some commands means they won't fail the build if they encounter errors.

### Stage 2: Runtime Stage
The second stage creates the final runtime image, starting fresh with PHP 8.3 FPM. This is separate from the builder stage, which means I only include what's necessary for running the application, not the build tools.

In this stage, I install only the essential packages I need for the application to run: Nginx as the web server and curl for health checks. I also install the same PHP extensions for database connectivity. Then I copy the entire prepared application from the builder stage into this runtime image. Since I'm only copying the finished application, the final image is much smaller than if I included all the build tools.

I create the `/app/var` directory for cache and logs, and I set the ownership to `www-data`, which is the user that the web server runs as. I set permissions to 755 for most directories and 775 for the var directory to allow the web server to write cache and log files. This prevents permission-related errors at runtime.

I then copy the Nginx configuration files that tell the web server how to handle requests. Finally, I expose port 8080, which is compatible with Railway's dynamic port assignment system. The startup command runs three important tasks: first it attempts to run database migrations to set up the database schema, then it starts PHP-FPM as a daemon process, and finally it starts Nginx in the foreground to keep the container running. The `|| true` after migrations means the container will still start even if migrations fail, which prevents the application from crashing if migrations have already been run.

---

## Part 2: Nginx Configuration (1-2 minutes)

### Overview
Nginx serves as the web server and reverse proxy for my application. It acts as the entry point for all requests, deciding whether to serve static files directly or forward PHP requests to PHP-FPM for processing.

### Server Configuration
The Nginx server listens on port 8080 and sets the document root to `/app/public`, which is Symfony's public directory. The default index file is set to `index.php`. I also configure logging to track all requests and errors, which is helpful for debugging production issues.

### Security Headers
I add several security headers to protect the application from common web attacks. The `X-Frame-Options` header prevents clickjacking attacks by restricting how the site can be framed. The `X-Content-Type-Options` header prevents browsers from trying to guess the content type of responses. The `X-XSS-Protection` header provides protection against cross-site scripting attacks.

### Static Asset Serving
When requests come in for static assets like CSS, JavaScript, and images in the `/assets/` directory, Nginx serves them directly without involving PHP. These responses include cache headers that tell browsers to cache the assets for one year. This means users won't have to re-download unchanged files, significantly improving performance and reducing server load.

### Security Rules
I implement several security rules to protect sensitive files. The `.well-known` directory is allowed for SSL certificate verification. All hidden files and directories starting with a dot are blocked from being accessed. Files ending with a tilde, which are typically backup files, are also blocked. These rules prevent accidental exposure of configuration files and other sensitive data.

### URL Rewriting for Symfony
When a request comes in for a regular URL, Nginx first checks if the file or directory exists. If it doesn't exist, it forwards the request to `index.php` with all the original query string parameters. This enables Symfony's routing system to work properly, allowing the framework to handle all URLs through a single entry point.

### PHP Request Handling
When Nginx receives a request for `index.php`, it forwards it to PHP-FPM running on port 9000. The `fastcgi_split_path_info` directive properly handles the PATH_INFO variable needed for routing. I include the standard FastCGI parameters and set the SCRIPT_FILENAME to the correct absolute path so PHP knows which file to execute. This communication between Nginx and PHP-FPM is what allows the web server to run PHP code.

---

## How They Work Together

When a user makes a request to the application, the request arrives at Nginx on port 8080. Nginx then checks if the request is for a static file. If it is, Nginx serves the file directly from the filesystem. If the request is for a PHP page or API endpoint, Nginx forwards it to PHP-FPM running on port 9000. PHP-FPM executes the Symfony application code and generates a response. This response is then sent back through Nginx to the user's browser.

This architecture provides several benefits. Nginx is much faster at serving static content than PHP, so separating static and dynamic content improves performance. PHP-FPM handles multiple concurrent requests efficiently. The separation also allows these components to be scaled independently if needed in the future.

---

## Key Points

The multi-stage build approach keeps the final Docker image as small as possible by including only runtime dependencies, not build tools. The Nginx and PHP-FPM combination provides better performance and scalability than using a built-in PHP development server. Proper file permissions ensure the web server can write cache and log files without encountering permission errors. The automatic database migrations run on container startup, so the database schema is set up before the application starts handling requests.

Security headers protect the application from common web attacks without adding complexity. The URL rewriting configuration enables clean Symfony routing with friendly URLs. Listening on port 8080 ensures compatibility with Railway's dynamic port assignment system. All of these design choices work together to create a production-ready containerized application that is secure, performant, and easy to deploy on Railway.

