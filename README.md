# 1 - Projects doc

*This is a personal (and not very organized yet) note that I use to remember my tech choices. It's used as a personal documentation too. Instead of writing one per projet in README.md, I write it here as there are more similarities than differences between my personal projects.*

# Documented projects
- PPW (personal website)
- PuzzleCook (cooking app)

# Architectures
**PPW**
- *Light Mono Architecture:* [Back + SPA] + SQLite

**PuzzleCook**
- *Strong Mono Architecture:* [Back + SPA] + PostgreSQL

# Repositories
For all solodev projects, I chose the **monorepo** architecture.
- It makes sense for a personnal project since  front and back are tightly coupled
- In development, it uses only 1 VS Code window, and 1 devcontainer
- When deploying automatically from a push, a single push can deploy both back and front.
- End to end tests are easier to run too.
- Note that both projects can be entirely put in subfolders, keeping clean file browsing.


# In local: dev containers

**Why?**
[Dev containers](https://code.visualstudio.com/docs/remote/containers) are the perfect way to develop for me.

**Pro's**
- Containerized projects avoid any interaction between packages globally installed. My computer stays clean over the time. I'm not afraid of installing weird stuff.
- Sandboxed environements do not mess 
	1. with the host OS packages
	2. between the different projects themselves. This implies you don't need tools like NVM or virtual env, since you have only one version of node or python installed. 
- You can be root on your machine, removing all permission-issues related
- For environment parity, it makes more sense to use a Linux in local than macOS or Windows.
- No matter if I'm on Windows or MacOS, it works exactly the same on any host OS thanks to docker
- As I use multiple computers (desktop + laptop), having dev containers ensures I have the exact same environment on other computers (packages, conf, etc.). If a work with another developer, or if i use another computer, a local installation takes only 1 command from VS Code: `Clone Repository in container volume` (note that this should be the only way to use a devcontainer, otherwise, performance is poor).
- I don't need to upgrade my packages so to say. I just re-build the container automatically.
- If I need to, I can inject environment variables at container build-time, which means I don't need to use a tool like foreman or dotenv before running my server in local. In return, it forces to rebuild the container when adding a new env var.
- Over my different projects, I can reuse common scripts doing the same things (installing OS-level packages, customizing my PS1, etc.)
- It's easier to work on iPad with vscode.dev / Github Codespaces

**Con's**
- General learning curve, there are definitely other things to know, for Remote Containers (and their configuration) and stuff related to Docker. Another layer to understand (networking, communication, etc.)
- Debug inside docker
- To get adequate performances, there is no other way than using `clone repository in container volume`. Keeping sync between your container and local volume will lead to performance and other issues. This means you don't have access to your code folder. However, you can easily download or upload file via VSCode, so it's an acceptable trade-off.

**How?**

`devcontainer.json` configuration
- For devcontainers, Microsoft provides its own images through [MCR](https://github.com/microsoft/containerregistry). unfortunately, they are still not compatible with ARM64 architecture. Moreover, they seem a little bit magic for some configuration. In the end, I chose to use official [Docker images](https://hub.docker.com/_/python) and configure them.
- On the containers, we choose to use `root` user. This removes a lot of hassle for configuration, and has no security implication, since containers have only one role anyway.
- To get a clean color prompts, and set aliases up, we use a custom `.bashrc` file that is copied in the container after creating it.

PPW
- 1 `Dockerfile`
- env is injected directly from `devcontainer.json` Note: To respect 12 factors better, we'd better use `runArgs` + `local.env` file to split env in separate file, but it does not work. Plus, it will make the build fail the first time because the `local.env` file does not exist.

PuzzleCook
- 2 `Dockerfile` (app + db) and 1 `docker-compose.yml` (docker-compose is used here to provide a clean way to have a separate Postgres DB installed. It will may be used for other tools like Redis in the future.)
- env comes from `local.env` file in `.devcontainer/`

**Running the local server**
- Back-end: `cd back; ./manage.py runserver`
- Front-end: `cd front; npm run dev`

**Access from another device**
Sometimes we need to access the website from a mobile device for dev purpose.

- For this to work, we need to ensure:
	- `/front/.env.local` file has a `VITE_BACKEND_HOST` that uses locale IP address, rather than localhost. 
	- `/back/.env.local` file has a `DJ_FRONT_HOST` that uses locale IP address, rather than localhost. 
> The right address can be be found running `ifconfig | grep 192` on the host machine. Note that the IP displayed when running vite server is the Docker one, so it won't work if I try to access it from outside.
- Note there is no need to run servers with special arguments, because:
	- vite runs server in both localhost and locale IP
	- and django server does not need to.
- Ports are automatically forwarded by VSCode.
- If it does not work, check this [VS Code settings](https://stackoverflow.com/a/67997839/2255491)

**Getting Data**
To get actual data using a production dump, it obviously work differently when the DB is PostGreSQL or SQLite.

- **PostGreSQL** (PuzzleCook)
We can access the database directly from the PG connection string provided by render. 
→ go to the db container, and from this container (not the app one), run `/scripts/dump_prod_to_local.sh` (crendentials may be asked) (this file has been copied to the db container during the build time)

- **SQLite** (PPW)
As the database in a file attached, we need to configure SSH first on Render, and then use `scp` command to copy the file to local.
→ run `.devcontainer/local_scripts/dump_prod_to_local.sh`, it should simply copy the SQLite file from  Render to local. Note than we will need the ssh passphrase to complete the operation.

*Sidenote : how SSH works on Render*
1. We need to [generate a public key](https://render.com/docs/ssh-generating-keys) (to do on any local computer)
2. Then we need to[ add this key to Render account](https://render.com/docs/ssh-keys). They key will be usable for any app.
3. Then, the SSH url to a particular app, can be found in the dashboard, here :
![image](/Users/dd/Library/Containers/co.noteplan.NotePlan3/Data/Library/Application Support/co.noteplan.NotePlan3/Notes/👨‍💻 Code/1 - My projects doc_attachments/B16CFCDC-B20E-49EA-A0D1-F9187416B200.png)
4. Note than using `scp` is [almost the same command](https://render.com/docs/disks#transferring-files) than `ssh`. WARN: scp command must NOT be run inside a ssh prompt, but inside a local prompt.

**IMPORTANT** : when rebuilding the repo from sources, we will need to copy the `id_ed25519` (in addition to env files) and re-run the building phase, sothat file this is copied to `/root/.ssh` folder.

**Other operations**
- **Data serialization:** we use 2 Django commands to serialize data in JSON. It helps to make some backups, or some migrations. However it can be error prone and hard to debug. Use carefully. 
- Dump **prod → local:** TODO 

# Environments
We try to respect [12 factors](https://12factor.net/) as possible (see [config section](https://12factor.net/config) in particular)

There are two ways of using env var in local environments:
- We can inject them into the environment at the container **build-time:**
	- 👍🏽 It does not require any additional tools like dotenv or config to inject these variables when the server runs then.
	- 👎🏽 The devcontainer build fails if the `.env` file is not present (which is always the case since this file is not versionned and we build the container from repository). it means we have to use the devcontainer recovery mode to inject `.env` file. 
	- 👎🏽 It forces us to rebuild the container for any change to a variable, which is way more painful than a server restart.
	- 👎🏽 It is error-prone since it's hard to know is the environment is actually synced with the env file.
- We can inject them when the process (server or db) starts at **runtime:**
	- 👍🏽 The devcontainer build does not fail if the `.env` file is not present. In that case, the failure will occur later when the server tries to start, which is easier to manage.
	- 👍🏽 There is no need to rebuild the container if a variable is changed, just restart the server
	- 👎🏽 It requires an additional tool (like dotenv, foreman, django-environ, etc.) or different command/config to run the server differently to inject this file.
	- 👎🏽 This slightly slows the server restart

**Choice**
- App service (in PuzzleCook and PPW) : **runtime** (variables need to be changed a lot)
- DB service (in Puzzlecook): **build-time** (ariables are not going to change)

**Files**
*We try to use the same filename across the project*
- `front` folder: Vite is automatically looking for a `.env.local` file to inject it
- `back` folder: django-environ is used, and `manage.py` has been changed to read from `.env.local` file.
- `.devcontainer/service_db` will contain `.env.local` for database (being in this directory shows it is used by the devcontainer itself, not the back code).

**Remote Render Host**
- for both back and front, variables are set in the render dashboard. Injection happens at build-time.

**Examples**
All env files have a  `<file>.example` versionned used to remember we need an env var to be setup.

# On internet: hosting on Render

**Pro's** 👍
- Render is chosen because it's a better heroku for almost everything: fast and clean dashbord, IaaS, free static websites, no buildpack, a subdirectory can be deployed, disks available, better pricing at scale, no toolbelt, up-to-date doc, many examples, secret files (not only dashboard), health check, etc.
- There is no buildpack (and it's a good thing). Build command is set entirely in the dashboard (we don't use a build script in repo, because it would not be 12factors friendly)
- Start command is not set in a ProcFile but directly in the dashboard, which is already more 12factors friendly.

**Con's** 👎
- Render has no release phase. This has weird implications:
	- in PuzzleCook: migrations are run in the *build* stage on PPW
	- in PPW: as disks are not mounted at build stage, there is no other choice than run migrations in start command!
- As a consequence, there is no pipeline workflow. Which means this is not ideal to make a deployment with multiple stages (staging, production).
- There is no true monorepo support. Which means both front and back apps gets actually the whole reco copied. This is not a real issue. It just forces us to run a `cd back` or `cd front` before running a command.
- No built-in CI for now, which can be very convenient.

# Static Files
Since render provides static sites for free and these sites are host on a performant CDN, in a monorepo, it's way more interesting to host assets (images, icons, etc.) in front, to benefit from this CDN.

For this purpose, assets must be moved to a `front/public` folder.
When running the build command, these files are copied "as is" along the `index.html` file and `assets` folder which contains automatically generated static files (like js and css).

# Tool choices

### Back-end / Django

**Package manager**
**[Poetry](https://python-poetry.org/)**
- Poetry is quickly becoming the default Python package manager to use.
- On the opposite, Pipenv has many issues (slower, bugs, etc.)
- With containerisation, back-end project is run without any virtualenv, which allows a neat simplification (there is no dinstinction between global and local packages, commands are run without prefixing anything, etc.)
- It's supported and advised by modern hosting tools like [Render](https://render.com/docs/deploy-django)

**DX tooling**
- [Black](https://black.readthedocs.io/) → No-brainer choice to speed up development. It is no beta anymore.
- [Pylance](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance) → Built-in Microsoft python tooling, evolving at a great pace
	- Extra paths can be added to ensure no issue with imports: https://github.com/microsoft/pylance-release/blob/main/TROUBLESHOOTING.md#unresolved-import-warnings (not used currently since Python repo is at root level)
	- Currently, to avoid duplicates messages with PyLint, we stick with Pylance linter only: https://github.com/microsoft/pylance-release/issues/818
- [django-browser-reload](https://github.com/adamchainz/django-browser-reload) to get hot reloading when using Tailwind with Django
- [django-watchfiles](https://github.com/adamchainz/django-watchfiles) faster file-watching than Django built-in, to have a more efficient hot reloading
- [ipython](https://ipython.org/) to get auto completion in shell
- [ipdb](https://pypi.org/project/ipdb/) to get ipython in a breakpoint (in that case, no need to install ipython)
- [django-extensions](https://github.com/django-extensions/django-extensions) to get Django auto-imports when running a shell via `shell_plus`
- [Flake8](https://github.com/PyCQA/flake8) a Python checker for quality. Pretty complementary to Linting (for example to be run in pre-commit hook and in CI)

**Misc**
- [whitenoise](http://whitenoise.evans.io/) is used to serve Django static files in production, otherwise it's not possible.
- [gunicorn](https://gunicorn.org/) is the server used for Django in production. We keep using runserver tool in local as it's more suited.
- [dj-database-url](https://github.com/jacobian/dj-database-url) to use string-like database config rather than dicts (NOTE: not used with SQLite)

### Realtime / websockets
With Django, I initally found two main ways to do that:
- Use [Django channels](https://github.com/django/channels) package. I'm not very convinved for these reasons:
	- It's a big change in Django, which changes how things work from the ground up.
	- I suspect packages I use are not compatible with the built-in Django dev server.
	- It forces changes at the hosting level too (for example: using pgbouncer) which makes config of a PaaS more complicated
	- In my opinion, async should be integrated to bare Django, not a installable package.
- Using an external pub/sub service like [Pusher](https://pusher.com/).
	- Good: very reliable, does one thing, but very well.
	- Good: additional services like "Presence" → no need to code them by yourself
	- Warn: pricing could become a turn-off if you have many users

In the meantime, I found a very promising alternative: [Soketi](https://docs.soketi.app/) which is an open source and self-hostable Pusher-like product. It has no usage limitations (except of course your own server), can scale, and is compatible with Pusher clients as it uses the same protocol. To be experimented.

### Vue

- Typescript is a no brainer-choice since it avoids many pitfalls of Javascript language.

**[Vite](https://vitejs.dev/)**
- Vite is next generation frontend tooling, and is one of the [official options for Vue 3](https://v3.vuejs.org/guide/installation.html#vite). It's main advantage is to use ES Modules that speeds Hot Module Replacement. It is recommended by Tailwind too (sponsoring it). It's made by the Vue team.
- I definitely want to avoid learning Webpack since it seems to be deprecated, and the JS community shifting towards tools like ESBuild.

**[Volar VS Code extension](https://github.com/johnsoncodehk/volar)**
- Recommended extension to use with Typescript

## CSS

**[Tailwind CSS](https://tailwindcss.com/)**
- No-brainer tool to ease CSS development (TODO: write a blog article about it!)
- setup guide optimized for vue3 with vite: https://tailwindcss.com/docs/guides/vue-3-vite



# Git
- There is one .gitignore file per project (.devcontainer, back, front). It's easier to manage that way rather than mixing everything.


# Other tests or previously used

### Heroku
Has severe limitations:
- No static sites => we have to use either the same app with django-SPA, or another app with a web server like express. In both cases, it adds useless complexity.
- No disks => we have to pay for a managed PostGresDB in all cases and can't use SQLite
- Buildpacks are painfull for mono repos (I had to put all files in the root, which was awful).
- The dashbaord is slow and the documentation is not up to date
- Generally speaking, no effort is made to improve the product for many years (no build, only run)

### Fly.io
Loved the idea to use Docker in production too, to get some parity with local but in practice:
- Does not respect 12factors at all, forced to version file tied to a specific environment
- It has a poor support (I did not succeed in something and had no response at all on the forum)
- Although the doc is very clean, it's very light. There are no examples.

### Django-SPA
It was used to allow to serve the SPA with the Django server, and allow having a single application in Heroku, instead of 2. It worked as expected, but is not necessary anymore, because Render serves SPAs as a static site for free, and performances are better since it's on a CDN. That way, the back-end server is not hit.
But it's definitely a tool to keep in mind for particular cases.
