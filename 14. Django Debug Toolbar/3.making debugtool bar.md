# Django Debug Toolbar Setup Transcript

> yes install it

- First, **install it** — you run this:

  ```bash
  uv add django-debug-toolbar
  ```

  While you do that, let me wire up the config. Let me read your current settings and root urls:

  Read 2 files

- Installing… let me wire the 4 pieces. **Step 1** — add it to `INSTALLED_APPS`:

- **Update**(`dashboard\settings.py`)
  ⌊ Added 1 line

  ```diff
    40      "django.contrib.messages",
    41      "django.contrib.staticfiles",
    42      "rest_framework",
  + 43      "debug_toolbar",
    44      "weather",
    45  ]
    46
  ```

- **Step 2** — add its middleware (right after SecurityMiddleware, as the docs require it early):

- **Update**(`dashboard\settings.py`)
  ⌊ Added 1 line

  ```diff
    46
    47  MIDDLEWARE = [
    48      "django.middleware.security.SecurityMiddleware",
  + 49      "debug_toolbar.middleware.DebugToolbarMiddleware",
    50      "django.contrib.sessions.middleware.SessionMiddleware",
    51      "django.middleware.common.CommonMiddleware",
    52      "django.middleware.csrf.CsrfViewMiddleware",
  ```

- **Step 3** — tell the toolbar which IPs may see it (only your own machine):

- **Update**(`dashboard\settings.py`)
  ⌊ Added 3 lines

  ```diff
    29
    30  ALLOWED_HOSTS = []
    31
  + 32  # Debug Toolbar only appears for requests coming FROM these IPs (your own machine).
  + 33  INTERNAL_IPS = ['127.0.0.1']
  ```

  ```diff
    37
    38  INSTALLED_APPS = [
  ```

- **Step 4** — give it a URL, but *only* when DEBUG is on (never in production):

- **Update**(`dashboard\urls.py`)
  ⌊ Added 8 lines

  ```diff
    16  """
    17
    18  from django.contrib import admin
  + 19  from django.conf import settings
    20  from django.urls import include, path
    21
    22  urlpatterns = [
    23      path("admin/", admin.site.urls),
    24      path("", include("weather.urls")),
    25  ]
  + 26
  + 27  # Debug Toolbar's own internal URLs — dev only.
  + 28  if settings.DEBUG:
  + 29      import debug_toolbar
  + 30      urlpatterns = [
  + 31          path("__debug__/", include(debug_toolbar.urls)),
  + 32      ] + urlpatterns
  ```

- **All 4 config pieces done.** ✅ Quick recap of what each does:

  | Step | Why |
  |---|---|
  | `INSTALLED_APPS` | turns the app on |
  | middleware | the toolbar hooks into every request here |
  | `INTERNAL_IPS` | only *your* machine sees it (never real users) |
  | `urls.py` (DEBUG only) | the toolbar's own internal pages, dev-only |

  ---

  **Now see it** (make sure `uv add django-debug-toolbar` finished first):

  1. Restart `runserver`
  2. Open your dashboard: http://127.0.0.1:8000/
  3. You should see a **toolbar tab on the right edge** of the page — click it to expand

  Load the page and tell me — **do you see the toolbar on the right?**

  Once it's there, I've got something specific I want to show you (the SQL panel — remember I promised you this one 🧐).

※ Cogitated for 48s

※**recap**: *We're finishing Phase 4 by installing Django Debug Toolbar. I've wired all four config pieces; you were running the install and restarting the server. Next: open the dashboard and tell me if the toolbar tab appears on the right edge. (disable recaps in /config)*
