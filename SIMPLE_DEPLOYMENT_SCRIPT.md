# Simple Deployment Configuration Script

## Part 1: Dockerfile Setup (1–2 minutes)

My Dockerfile builds a Docker container for my Symfony app. It has two parts: a builder and a runtime.

**Builder Stage:**
I start with PHP 8.3 and install tools I need: git, npm, and databases packages. I install Composer to manage PHP libraries. I copy my project files and install all dependencies. I also warm up the cache to make the app faster.

**Runtime Stage:**
I create a new clean image with just PHP and Nginx. Nginx is the web server that handles requests. I copy my prepared app from the builder stage. I set up folders and permissions. Then I start PHP-FPM and Nginx when the container runs.

**Why two stages?**
The builder stage has lots of build tools. I don't need these in production, so I throw them away. The final image is much smaller.

**Startup:**
When the container starts, it runs three things: database migrations, PHP-FPM, and Nginx. If migrations fail, the container still starts.

---

## Part 2: Nginx Configuration (1–2 minutes)

Nginx is the web server. It listens on port 8080 and decides what to do with each request.

**Static Files:**
If you ask for CSS, images, or JavaScript files, Nginx serves them directly. These files are cached for one year in the browser.

**Security:**
I add special headers to protect against attacks. I block hidden files and backup files. I allow `.well-known` for SSL certificates.

**Routing:**
When you visit a URL, Nginx checks if it's a real file. If not, Nginx sends it to `index.php`. This is Symfony's main entry point.

**PHP Requests:**
When Nginx gets a PHP request, it forwards it to PHP-FPM. PHP-FPM runs the code and sends the response back.

**Flow:**
Request → Nginx → Static file? Serve directly → PHP file? Send to PHP-FPM → Response → Back to user

---

## Why This Works

- **Nginx is fast** at serving files
- **PHP-FPM handles** the app logic
- **Two stages** make the image smaller
- **Security headers** protect the app
- **Port 8080** works with Railway
- **Auto migrations** set up the database

That's it! Simple setup for a production app.
