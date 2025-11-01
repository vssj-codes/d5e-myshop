# Project run steps — keep WeasyPrint / Celery working (macOS M1, bash)

Save this file in your project root and follow it when starting the project.

---

## One-time setup (do this once per machine)

1. Install system deps (Homebrew):
```bash
brew update
brew install pkg-config cairo pango gdk-pixbuf libffi glib gobject-introspection libxml2 libxslt
````

2. (If `weasyprint` complains about gobject) Create a small symlink so the loader can find the lib:

```bash
ln -s "$(brew --prefix)/lib/libgobject-2.0.dylib" "$(brew --prefix)/lib/libgobject-2.0-0.dylib"
```

3. Add Homebrew/pkconfig paths and the macOS fallback library path to your **bash** startup so new terminals pick them up:

```bash
# append to ~/.bash_profile
echo 'export PATH="/opt/homebrew/bin:$PATH"' >> ~/.bash_profile
echo 'export PKG_CONFIG_PATH="$(brew --prefix)/lib/pkgconfig:$PKG_CONFIG_PATH"' >> ~/.bash_profile
echo 'export LDFLAGS="-L$(brew --prefix)/lib $LDFLAGS"' >> ~/.bash_profile
echo 'export CPPFLAGS="-I$(brew --prefix)/include $CPPFLAGS"' >> ~/.bash_profile
echo 'export DYLD_FALLBACK_LIBRARY_PATH="$(brew --prefix)/lib:$DYLD_FALLBACK_LIBRARY_PATH"' >> ~/.bash_profile

# load now
source ~/.bash_profile
```

4. Persist the macOS fallback path into the virtualenv activation script so any shell that activates the venv (including VS Code terminals) gets it:

```bash
echo 'export DYLD_FALLBACK_LIBRARY_PATH="$(brew --prefix)/lib:$DYLD_FALLBACK_LIBRARY_PATH"' >> env/myshop/bin/activate
```

5. (Re)install WeasyPrint inside the project virtualenv:

```bash
source env/myshop/bin/activate
pip uninstall -y weasyprint
pip install --no-cache-dir weasyprint
python -m weasyprint --version   # should print a version like "WeasyPrint version 66.0"
deactivate
```

---

## Daily start (every time you work on the project)

Open three terminals (or editor terminals) and follow the order below.

1. **Terminal 1 — RabbitMQ (Docker)**
   (From project root `for-video/` — adjust if you use docker-compose)

```bash
# either compose
docker-compose up -d rabbitmq

# or plain docker
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

2. **Terminal 2 — Django dev server**

```bash
# from project root (for-video/)
source env/myshop/bin/activate
cd myshop
python manage.py runserver
```

3. **Terminal 3 — Celery worker**

```bash
# from project root or from myshop/, just ensure you activate the venv using the correct relative path
# if you are inside for-video/myshop:
deactivate || true
source ../env/myshop/bin/activate
cd myshop               # if not already inside
python -m celery -A myshop worker -l info
```

**Notes:**

* Use `python -m celery` to avoid subtle wrapper differences.
* Ensure the venv you activated is the same one that contains `weasyprint` (check `which python` and `which celery` if in doubt).

---

## Quick checks / troubleshooting

If something fails, run these in the **same shell** where you plan to start celery:

```bash
# confirm env & libs
echo "DYLD_FALLBACK_LIBRARY_PATH=$DYLD_FALLBACK_LIBRARY_PATH"
which python
which celery
python -c "import weasyprint; print('weasyprint OK', getattr(weasyprint,'__version__','?'))" || python -c "import traceback; traceback.print_exc()"

# If weasyprint import fails, try the symlink + re-source venv activate:
ln -s "$(brew --prefix)/lib/libgobject-2.0.dylib" "$(brew --prefix)/lib/libgobject-2.0-0.dylib" || true
deactivate || true
source env/myshop/bin/activate
python -m weasyprint --version
```

If Celery still crashes while Django imports `weasyprint` at startup:

* Confirm the same venv is active in the Celery terminal (`which python` should point to `.../env/myshop/bin/python`).
* Start Celery with `python -m celery ...` (not the shell wrapper).
* Ensure `DYLD_FALLBACK_LIBRARY_PATH` is present in that shell (`echo $DYLD_FALLBACK_LIBRARY_PATH`).

---

## Helpful tips

* In VS Code set the integrated terminal shell to **bash** and enable login/profile reading (so `~/.bash_profile` runs).
* Prefer `python -m celery` to avoid wrapper script oddness.
* If you move to CI or Docker, replicate the system libs in the container (install `cairo`, `glib`, `pango`, etc.) or use a WeasyPrint-ready image.
* Keep `env/myshop/bin/activate` updated with the `DYLD_FALLBACK_LIBRARY_PATH` line so all shells that activate the venv get the correct path automatically.

---

## Minimal test to confirm everything works

1. Activate venv and test weasyprint:

```bash
source env/myshop/bin/activate
python -m weasyprint --version
```

2. Quick Celery test (from `myshop/`):

```bash
# start worker in terminal 3
python -m celery -A myshop worker -l info

# in another terminal with venv activated (project root or myshop)
python manage.py shell -c "from orders.tasks import test_weasyprint_pdf; print(test_weasyprint_pdf.delay())"
```

Then check for `tmp/weasy_test.pdf` in project and Celery logs for success.

---

If anything changes (macOS update, Homebrew prefix change), re-run the One-time setup steps.

```