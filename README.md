# etesync-local-dev-setup-guide
A comprehensive guide to setting up EteSync-Server and EteSync-Web for local development on WSL2 using Daphne and Caddy, including solutions to common pitfalls.

# EteSync Local Development Setup Guide (WSL2, Python, Caddy)

## Project Goal & Motivation

This project was born out of a strong desire to organize my time and tasks using a personal calendar and task management system, while crucially maintaining **complete control and privacy over my data**. The EteSync ecosystem, with its self-hostable server and web client, was the perfect fit for this.

The journey to get it running locally on my WSL2 (Windows Subsystem for Linux 2) Ubuntu environment proved to be a significant challenge. What started as a seemingly straightforward setup evolved into a deep dive into Python backend debugging, ASGI server intricacies, and complex Caddy reverse proxy configurations. Every obstacle, from elusive recursion errors to stubborn SSL issues, tested my resolve.

However, the determination to make it work for personal privacy and local data storage paid off! This guide is the culmination of that effort, designed to provide a clear, step-by-step path for anyone else who wishes to set up their EteSync calendar and task server locally. My aim is for this documentation to be so precise that you can follow it and get your own setup working seamlessly, avoiding the many pitfalls I encountered.

---

## Prerequisites

Before you begin, ensure you have the following installed and configured within your WSL2 Ubuntu distribution:

* **WSL2 with Ubuntu:** Your Linux environment for running the backend and Caddy.
* **Python 3.8+:** Available in your WSL2 Ubuntu.
* **Node.js and npm/yarn:** For building the EteSync-Web frontend.
* **`git`:** For cloning the repositories.
* **`daphne`:** The ASGI server for the EteSync backend.
* **`caddy`:** The powerful reverse proxy.
* **Basic Linux Command Line Knowledge:** Familiarity with `cd`, `ls`, `cat`, `nano` (or `vim`), `sudo`.
* **Web Browser:** Chrome, Brave, Firefox, or Edge for testing.

---

## Section 1: Initial Setup & Project Cloning

First, let's get the necessary project repositories onto your system. We'll assume you'll clone them into a `projects` directory within your WSL2 home folder.

1.  **Open your WSL2 Ubuntu terminal.**
2.  **Create a `projects` directory (if it doesn't exist) and navigate into it:**
    ```bash
    mkdir -p ~/projects
    cd ~/projects
    ```
3.  **Clone the EteSync-Server repository:**
    ```bash
    git clone [https://github.com/etesync/etesync-server.git](https://github.com/etesync/etesync-server.git)
    ```
4.  **Clone the EteSync-Web repository:**
    ```bash
    git clone [https://github.com/etesync/etesync-web.git](https://github.com/etesync/etesync-web.git)
    ```

Now you should have two directories: `~/projects/etesync-server` and `~/projects/etesync-web`.

---

## Section 2: EteSync-Server Backend Setup

This section covers setting up the Python backend, including its dependencies, database, and preparing it to run with Daphne.

1.  **Navigate to the EteSync-Server directory:**
    ```bash
    cd ~/projects/etesync-server
    ```
2.  **Create a Python virtual environment and activate it:**
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate
    ```
    (You should see `(.venv)` at the start of your prompt, indicating the virtual environment is active.)
3.  **Install backend dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
4.  **Navigate into the core `etebase_server` application directory:**
    ```bash
    cd etebase_server
    ```
5.  **Perform Django database migrations:**
    ```bash
    python manage.py migrate
    ```
6.  **Create a Django superuser (optional, but recommended for admin access):**
    ```bash
    python manage.py createsuperuser
    ```
    Follow the prompts to create a username, email, and password.

### **Problem & Solution: "maximum recursion depth exceeded" during Signup**

**Problem Description:**
When attempting to sign up a new user via the frontend, the browser displayed `"maximum recursion depth exceeded"`. Daphne's logs only showed a `400 Bad Request` without a detailed Python traceback, indicating the application was catching and suppressing the error. This pointed to an infinite recursion loop within the Python backend's user creation logic.

**Diagnosis:**
1.  **Initial `pdb` trace:** We added `import pdb; pdb.set_trace()` to the `signup_save` function in `etebase_server/fastapi/routers/authentication.py`.
2.  **Stepping with `pdb`:** By using `s` (step into) and `n` (next line) in the `pdb` prompt, we traced the execution flow into `etebase_server/django/utils.py` to the `create_user` function.
3.  **Identifying the loop:** The `pdb` output revealed that `create_user` was calling `custom_func`, and `custom_func` was itself configured to be `create_user`, leading to an infinite recursive call.
    ```python
    # Snippet from etebase_server/django/utils.py (where the loop was identified)
    def create_user(context: CallbackContext, *args, **kwargs) -> UserType:
        custom_func = app_settings.CREATE_USER_FUNC
        if custom_func is not None:
            return custom_func(context, *args, **kwargs) # <-- This was calling create_user itself!
        # ... (rest of the default create_user logic)
    ```
4.  **Root Cause:** The `app_settings.CREATE_USER_FUNC` was dynamically loaded from Django's `settings.py`, where it was incorrectly set to point back to `etebase_server.django.utils.create_user`.

**Solution:**
Modify `settings.py` to set `ETEBASE_CREATE_USER_FUNC` to `None`. This tells the `create_user` function to use its default, non-recursive logic.

1.  **Open `settings.py` for editing:**
    ```bash
    nano ~/projects/etebase/etebase_server/settings.py
    ```
2.  **Find the line:**
    ```python
    ETEBASE_CREATE_USER_FUNC = "etebase_server.django.utils.create_user"
    ```
3.  **Change it to `None` (or comment it out):**
    ```python
    # ETEBASE_CREATE_USER_FUNC = "etebase_server.django.utils.create_user" # Comment out this line
    ETEBASE_CREATE_USER_FUNC = None # Or set it to None, which is safer
    ```
4.  **Save the file.**
5.  **Clean up any debugging lines (important!):**
    * Remove `import pdb; pdb.set_trace()` from `~/projects/etebase/etebase_server/fastapi/routers/authentication.py`.
    * Remove `import sys` and `sys.setrecursionlimit(...)` from the top of `authentication.py`.

---

## Section 3: EteSync-Web Frontend Setup

This section covers setting up and building the EteSync-Web frontend.

1.  **Navigate to the EteSync-Web directory:**
    ```bash
    cd ~/projects/etesync-web
    ```
2.  **Install frontend dependencies (using npm or yarn):**
    ```bash
    npm install # Or 'yarn install' if you prefer yarn
    ```
3.  **Build the frontend for production:**
    This command compiles the React application into optimized static HTML, CSS, and JavaScript files, typically placed in a `build/` directory.
    ```bash
    npm run build # Or 'yarn build'
    ```
    After this, you should see a `build/` directory created inside `~/projects/etesync-web/`. This `build/` directory is what Caddy will serve.

---

## Section 4: Caddy Reverse Proxy Configuration

Caddy will serve your EteSync-Web frontend and proxy API requests to your EteSync-Server backend, handling HTTPS automatically.

1.  **Install Caddy (if not already installed):**
    Refer to the official Caddy documentation for installation instructions for your WSL2 Ubuntu distribution: [https://caddyserver.com/docs/install](https://caddyserver.com/docs/install)

2.  **Create your `Caddyfile`:**
    Navigate back to your `etesync-server` project root (where you cloned the repo) as this is a convenient place to keep the `Caddyfile`.
    ```bash
    cd ~/projects/etebase/etebase_server # Go back to the etebase_server directory
    nano Caddyfile # Or 'vim Caddyfile'
    ```

### **The Final, Working `Caddyfile`**

This `Caddyfile` configuration successfully handles all proxying, SSL, and routing for your local development setup.

**Paste this exact content into your `Caddyfile`:**

```caddyfile
{
    debug # Enable debug logging for verbose output (useful for troubleshooting)
}

localhost:8443 {
    # Configure TLS for localhost using Caddy's internal CA.
    # This automatically generates and trusts a certificate for development,
    # preventing ERR_SSL_PROTOCOL_ERROR and avoiding binding to port 80.
    tls internal

    # --- 1. Handle API Requests (Highest Priority) ---
    # This block specifically matches requests for the EteSync API.
    # The 'handle' directive ensures that if the path matches /api/api/*,
    # Caddy will ONLY execute the reverse_proxy directive within this block
    # and then stop processing this request. This prevents API calls
    # from accidentally being treated as static files.
    handle /api/api/* {
        # Proxy the request to the Daphne server running on 127.0.0.1:8001.
        reverse_proxy [http://127.0.0.1:8001](http://127.0.0.1:8001)
    }

    # --- 2. Serve Static Frontend Files and SPA Fallback ---
    # This 'handle' block acts as a catch-all for any requests that did NOT
    # match the /api/api/ handle block above. This is where your EteSync-Web
    # static files and Single Page Application (SPA) routing logic belong.
    handle / {
        # Set the root directory for serving static files to your EteSync-Web build output.
        # IMPORTANT: Ensure this path is correct for your system.
        root * /home/user/projects/etesync-web/build
        
        # Enable Caddy's file server to serve files from the root directory.
        file_server

        # For SPAs, if a specific file is not found (e.g., a client-side route like /settings),
        # serve index.html instead. This allows the frontend's JavaScript router to handle the route.
        try_files {path} /index.html
    }

    # Configure logging to stdout in JSON format (useful for debugging).
    log {
        output stdout
        format json
    }
}
```

### **Understanding the Caddyfile Evolution & Problem Solving:**

Throughout the debugging process, we encountered several common issues related to Caddy's configuration and its interaction with the backend and frontend. Here's a breakdown of the problems and their solutions, which are integrated into the final `Caddyfile`:

* **Problem: `502 Bad Gateway` (Caddy couldn't reach Daphne)**
    * **Diagnosis:** Caddy logs showed `502` errors, indicating it couldn't establish a connection to the backend Daphne server. This often happened if Daphne was not running, or if there were network accessibility issues within WSL2.
    * **Solution:**
        * Ensured Daphne was always running (`daphne -b 0.0.0.0 -p 8001 etebase_server.asgi:application`) before starting Caddy.
        * Confirmed Daphne's direct accessibility from within WSL2 using `curl http://127.0.0.1:8001/api/api/v1/authentication/signup/` (which should return a `422` or `400` from FastAPI, confirming connectivity).

* **Problem: `404 Not Found` for API calls (FastAPI prefix mismatch)**
    * **Diagnosis:** Caddy successfully proxied requests to Daphne, but Daphne's logs showed `404` for API endpoints. This occurred because the FastAPI application in `etebase_server/fastapi/main.py` was being initialized without a `prefix` argument in `etebase_server/asgi.py`. Consequently, FastAPI expected API paths like `/v1/...` relative to its root, but Caddy was forwarding the full path including the initial `/api` (e.g., `/api/api/v1/...`).
    * **Solution:** Modified `etebase_server/asgi.py` to pass `prefix="/api"` to the FastAPI `create_application` function. This aligns FastAPI's internal routing with the path Caddy forwards.
        ```python
        # etebase_server/asgi.py
        def create_application():
            from etebase_server.fastapi.main import create_application
            app = create_application(prefix="/api") # <-- The fix!
            app.mount("/", django_application)
            return app
        ```

* **Problem: `ERR_SSL_PROTOCOL_ERROR` / `bind: permission denied` on port 80**
    * **Diagnosis:** The browser displayed SSL protocol errors when trying to access `https://localhost:8443/`. Caddy logs showed "permission denied" errors when attempting to bind to privileged port 80 (the standard HTTP port) for automatic HTTP-to-HTTPS redirects. This happened even when only HTTPS was explicitly configured, as Caddy's default `auto_https` behavior tries to manage both HTTP and HTTPS.
    * **Solution:**
        * Used the `tls internal` directive within the `localhost:8443` site block in the `Caddyfile`. This explicitly tells Caddy to manage TLS for `localhost` using its internal Certificate Authority, without needing to bind to port 80 for redirects.
        * Running Caddy with `sudo` (`sudo caddy run ...`) was necessary to grant it the required permissions for other potential system-level operations (though `tls internal` aims to minimize the need for privileged ports).

* **Problem: API requests serving `index.html` (Caddy directive order/`try_files` precedence)**
    * **Diagnosis:** Caddy's debug logs revealed that `POST /api/api/...` requests were being rewritten to `/index.html` by the `try_files` directive. This meant the frontend's `index.html` was being served instead of the API response, and the backend (Daphne) never received the API request. This occurred because `try_files` (a rewrite handler) can sometimes take precedence over `reverse_proxy` or act as a fallback even for non-GET requests if not carefully isolated.
    * **Solution:** The final `Caddyfile` employs two distinct `handle` blocks, leveraging Caddy's explicit directive ordering:
        * The `handle /api/api/*` block is placed first. When a request matches this path, Caddy processes *only* the `reverse_proxy` directive within it and then stops. This ensures API requests are always proxied.
        * The subsequent `handle /` block acts as a catch-all for all *non-API* requests. Within this block, `root`, `file_server`, and `try_files` correctly serve the static frontend assets and manage client-side routing for the Single Page Application.

---

## Section 5: Running the Services & Final Verification

Now that everything is configured, let's start the services and verify the setup.

1.  **Open two separate WSL2 Ubuntu terminal windows.**

2.  **In Terminal 1 (Daphne):**
    * Ensure your Python virtual environment is active: `source .venv/bin/activate` (if not already).
    * Navigate to the `etebase_server` directory: `cd ~/projects/etesync-server/etebase_server`
    * Start Daphne:
        ```bash
        daphne -b 0.0.0.0 -p 8001 etebase_server.asgi:application
        ```
    * Leave this terminal open.

3.  **In Terminal 2 (Caddy):**
    * Navigate to the directory where you saved your `Caddyfile`: `cd ~/projects/etesync-server/etebase_server`
    * Start Caddy (you will be prompted for your WSL2 password):
        ```bash
        sudo caddy run --config Caddyfile --adapter caddyfile
        ```
    * Leave this terminal open.

4.  **Open your web browser (Chrome, Brave, Firefox, Edge) on your Windows machine.**

5.  **Navigate to `https://localhost:8443/`**
    * You should now see the EteSync-Web frontend load correctly.

6.  **Attempt to sign up a new user.**
    * Fill in the required fields (username, password, etc.).
    * Click the signup button.
    * **Expected Result:** The signup process should complete successfully without the "maximum recursion depth exceeded" error. You might get other validation errors if your input isn't fully compliant with EteSync's API, but the core setup should now be working.
    * **Check logs:** Observe both the Caddy and Daphne terminals. You should see Caddy logging the `POST` request to `/api/api/v1/authentication/signup/` and showing a successful proxy to Daphne. Daphne should then log the incoming request and process it.

---
