---
title: Building with Claude Code: A FamilyOffice app written in FastAPI and React 
categories:
- Tech
feature_image: "https://picsum.photos/2560/600?image=872"
published: false
---

Public repo on [Github](https://github.com/ajaythomas/family_office/)

For a while now, I have watched my dad painstakingly use a combination of Excel, handwritten notes and Finance watchlists to manage his household's investments. This was a good excuse to build a hobby app with Claude Code. It gave me the opportunity to experiment with a few technologies I wanted to experiment with:

1. FastAPI for the Python backend
    1. uv for Python package management
    1. Alembic for database migrations
1. React for the frontend and Vite (serving as the frontend build tool)
1. Docker containers for local and production deployment
1. Cedar as an authorization policy language (for defining roles and permissions of my app's end users)
1. OAuth 2.0 for Google Calendar integration

I also didn't want to be knee deep in any cloud platform ecosystem, so to productionize the app - I used the Hetzner cloud vs managed AWS offerings or other app platforms (Railway, Render, Vercel etc.)

I captured some of my learnings below:

## Python ecosystem

I had used Django in the past and it was a pleasant upgrade using FastAPI especially for a simple hobby app like this. FastAPI also meant I could also use SQLAlchemy (for ORM) and the lighter alembic (SQLAlchemy-compliant tool) vs the built-in Django migrations for their ORM. I continued to use MyPy for static type checking.

The uvicorn ASGI (Async Web Server Gateway Interface) web server was also a good change compared to uWSGI (I used at my previous job) whose workers could occasionally be overloaded by requests. Concurrency is better managed by ASGI which handles thousands of simultaneous connections asynchronously, while uWSGI handles one request per worker thread, making it less efficient under high concurrency. Again, I can't honestly compare a hobby app's request load against a production grade uWSGI app at my previous job, so YMMV.

uv as a Python package manager is a lot cleaner and readable and faster than pip and its associated tools. [pyproject.toml](https://github.com/ajaythomas/family_office/blob/main/pyproject.toml) is the readable dependency ledger (similar to pip's `requirements.txt`) while uv.lock gives the entire transitive tree of dependencies. Both are good to merge to a public repo. uv.lock ensures deterministic builds every time you run docker compose up - so you get the exact same versions of every package. Without it, a sub-dependency could update overnight and break your build. Also, I don't upgrade dependencies mentioned in my uv.lock unless absolutely needed so my dependency supply chain is not just pulling in every random update even if I don't need it.

PyPI is the Python software repo - where dependencies are installed from when you run `uv install` for the various pyproject.toml dependencies. Similar to npm for NodeJS. If you accidentally installed a dependency and no longer require it, you’ll see it in the pyproject.toml and uv.lock. Just run `uv remove authlib` and uv remove updates both pyproject.toml and uv.lock in one step.

APScheduler was my cron jobs framework (to send out calendar events) that lived inside the same FastAPI container.

A clean split in my app backend:

1. schemas.py for the API request/response shape (inheriting from BaseModel i.e. Pydantic’s validation/model class)
1. models.py for the db table definitions

Regarding db models/tables, I recently learnt about:

1. `mapped_column` ([example](https://github.com/ajaythomas/family_office/blob/main/app/models.py#L34)), serving as the modern replacement for the traditional Column construct when using the SQLAlchemy ORM SQLAlchemy ORM. Because it works with Python type hints, it provides better IDE support and static type checking. It automatically derives database types and nullability from Python type annotations used with Mapped[]. For example, `Mapped[int]` implies `nullable=False`, while `Mapped[Optional[int]]` implies `nullable=True`.
1. SQLAlchemy ORM's `autocommit` and `autoflush` params when calling [sessionmaker](https://github.com/ajaythomas/family_office/blob/main/app/database.py#L17C16-L17C28).
    1. `autocommit=False`: This is the default and recommended setting. It ensures all operations happen within a transaction. You must explicitly call session.commit() to save changes or session.rollback() to discard them.
    1. `autoflush=False`: Disables "automatic" flushing. Normally, SQLAlchemy sends pending changes to the database right before you issue a query so the results are accurate. With this set to False, you must manually call session.flush() if you need the database to process changes (like generating an ID) before the final commit.

### Login

I added a simple google login using Google Identity Services (@react-oauth/google) that follows the OIDC implicit flow. To support this, I created an app in Google Cloud Console and on the screen `OAuth 2.0 client config screen > Authorized Javascript Origins`, I added the Vite frontend (local testing: http://localhost:5173 and whatever prod equivalent) which is where `@react-oauth/google` JavaScript runs. Google validates the origin of the page that initiates the sign-in flow, which is the browser loading the React app, not the FastAPI server (that listens on port 8000). Additionally, the Google app's client ID is specified in my `.env` file and used by my app to verify the Google ID token after the OIDC flow completes.

### Logging

For logging, I used Python's built-in logging module by adding this to each file:
`import logging logger = logging.getLogger(name)`
Then, used it like so:
`logger.info("User %s logged in", user.email) logger.error("Portfolio creation failed for user %s", user.id)`
To read the logs, I would stream them from my app docker container:
`docker compose logs -f app`
An interesting thing I learnt about Python f-strings vis-a-vis logging was that even if f-strings are the modern Python convention for general string formatting; for logging - the `%s` style is preferred. Because the logger defers formatting until it knows the message will actually be emitted. If you wrote `logger.info(f"User {user.email} logged in")`, the f-string gets evaluated immediately regardless of log level, so you pay the formatting cost even when the log is filtered out. Everywhere else in the codebase, f-strings are the right call.

### Cedar Authorization

One of the primary reasons to do this app was to learn more about the new-ish authorization language (released 2021) from AWS called Cedar. They have a managed version but I just used the opensource python SDK within my FastAPI app. Having worked in authorization for sometime, it was refreshing to use a simple readable language like Cedar that lets me create RBAC (role-based access control) and ABAC (attribute based access control) fairly easily. Though supporting ReBAC using Cedar felt forced, it was not a need for my app. ReBAC is better supported by Zanzibar-like languages namely [SpiceDB](https://authzed.com/spicedb) and [OpenFGA](https://openfga.dev/).

Check out [Cedar's simple, short tutorial](https://cedarpolicy.com/en/tutorial/).

The Cedar engine makes fast authorization decisions using:

1. Data about principal and resource entities
1. Additional context needed like IP address, user agent etc.
1. Policy inventory i.e. all the permit and forbid policies matching against one or more principal, actions or resources specified in requests

For my app, I defined [my Cedar policies and schema here](https://github.com/ajaythomas/family_office/tree/main/app/policies). This is then used ([example](https://github.com/ajaythomas/family_office/blob/main/app/routers/portfolios.py#L76)) throughout my app via an [authorize() method I defined](https://github.com/ajaythomas/family_office/blob/main/app/cedar_authz.py#L29).

## VSCode IDE and plugins

1. Claude Code
    1. Claude Code on VSCode and ClaudeCode on terminal use the same memory but two separate conversation sessions. Each Claude Code session (terminal, VS Code extension, etc.) is independent with its own conversation context. They don't share messages or see each other's chat history. The shared memory files in ~/.claude/projects/ persist across sessions, so facts Claude saves there are available to both — but the actual conversation thread is separate in each interface.
1. autoDocstring - python docstring generator
1. Cedar (for my authz policies)
1. Code Spell Checker
1. Microsoft's tools like Container Tools, MyPy Type Checker, PostgresSQL (which I used to connect to both localhost and Hetzner cloud postgres db docker containers)
1. Ruff

## OAuth: Google Calendar

Along the same vein of the [login flow mentioned above](#login), Google Calendar integration (to support adding earnings dates of your stock tickers to your gCal) needs OAuth integration. For that:
Go back to your app in the Google Cloud Console, and add the below endpoint to `Authorized redirect URIs`:
`http://localhost:8000/auth/google-calendar/callback` (for local testing or a similar api endpoint in prod): This is needed for Google Calendar OAuth 2.0 authorization code flow. Google redirects the browser to our registered callback URL with an auth code.
This is different from the Google login which uses Google Identity Services (@react-oauth/google) that follows the OIDC implicit flow. The login flow returns the signed ID JWT token directly to a JavaScript callback in the browser — no redirect URI involved. Google never redirects to the server; our frontend just calls `POST /auth/google` with the token it received. That's why no redirect URI is needed in Cloud Console for the login flow.

A few more additional steps are mentioned in [my project's README](https://github.com/ajaythomas/family_office/tree/main#google-calendar-integration).

## Production Deployment (and some docker tips)

I wanted to be cloud agnostic so decided against a managed ecosystem like AWS lambda with managed Postgres or similar Azure setups. Because, I already had docker containers deployed locally, it was not much of a lift to deploy the same containers on an EC2 instance or in my case, a bare-bones Hetzner cloud instance. Based in EU, I was impressed with the lightweight and cheap instance, I was able to provision ($4 monthly).

Details are mentioned [in my README](https://github.com/ajaythomas/family_office/tree/main#production-deploy-hetzner)

I could SSH into my prod docker containers easily with SSH keys I setup for my Hetzner server. Setting up aliases in my bashrc file to SSH into the prod server and prod db and postgres connection from my VSCode IDE were some QoL improvements.

### Docker Profiles

Take a look at [my docker compose file](https://github.com/ajaythomas/family_office/blob/main/docker-compose.yml#L16). Only the db has no `profiles` attribute meaning it is the default profile. Other containers do.
Docker profiles lets me control what starts by default (i.e. in local deployment, only the postgres docker container was needed). For prod, by specifying a separate prod profile, additional to postgres, the profile also container `api`, `web`, `caddy` containers too.
So `docker compose up -d` in dev still starts only the DB which bound to port 127.0.0.1. Whereas, on my Hetzener server, I ran
`COMPOSE_PROFILES=prod docker compose up -d --build` which told docker to run the prod profile.
Note, the `--build` flag tells Compose to build images from the Dockerfiles before starting containers. Without --build it would look for pre-built images, which don't exist since we haven't pushed these containers to a registry.

This strategy of docker profiles saved me the trouble of pushing a separate `deploy` branch (with these containers) to my repo remote in addition to `master`. More of my learnings from Docker are captured in [my notes](https://github.com/ajaythomas/family_office/tree/main/notes#docker).

### Caddy

For my prod deploy, I used [Caddy](https://caddyserver.com/), a free reverse proxy for my self hosted (on Hetzner Cloud) deployment that automatically provisions SSL certificates for my domain too. Caddy file [looks like so](https://github.com/ajaythomas/family_office/blob/main/Caddyfile).

## Claude Code

ClaudeCode (using Sonnet 4.6) was a good partner in building this project. First, I worked with it to create [a plan](https://github.com/ajaythomas/family_office/blob/main/plan.md). With some back and forth, we landed on an architecture, what to build vs use OSS vs use vendors etc.

I defined a [Claude.md file](https://github.com/ajaythomas/family_office/blob/main/CLAUDE.md) that is read by Claude Code on each prompt. Keep it short under 200 lines so you're not wasting too many tokens.

Both the plan and claude.md are living files which you continuously update through phases. 

I had a [clear exit criteria](https://github.com/ajaythomas/family_office/blob/main/plan.md#build-order) for each phase - running tests, human in the loop for seeing live demo and commit to remote before Claude Code moved to the next phase.

Sometimes, it jumped to wrong conclusions or did narrow fixes:

1. When I asked it to update the gain/loss column to be percentage instead of absolute dollar amount, it made the change in FE vs in the portfolios.py class' method `get_portfolio()`. BE changes are always preferred so if a mobile client later introduced, it gets that too.
1. When working on `cedar_authz.py`, it saw `cedarpy.isauthorized()` fail because of type errors and assumed it was broken on Python 3.14. But, `cedarpy` docs clearly says 3.14 is supported. It tried to downgrade my Python version (insane!) After I questioned it, it agreed that it jumped to a wrong conclusion. When I explicitly asked it to look at the principal/action/resource triplet it was sending to that method and try different variations of the parameters, it landed on a working cedar authorization call. I had to rein it back before it started downgrading my python versions.

Where I really liked its support was the security review agent I ran before making my repo public. It caught a few holes:

1. Weak password for my db mentioned in the docker-compose.yml file (that is NOT gitignored). Even if the db is exposed only to localhost (for both dev and prod; and prod needs me to SSH into it); better to use an env var for the same.
1. An OAuth CSRF vuln because the state parameter was just using the user's ID.

## FrontEnd

Frontend is new territory for me. I used a lot of Claude Code to get this up. Learnings captured in [my notes](https://github.com/ajaythomas/family_office/tree/main/notes#frontend).

## Outro

Overall, it was good to play with Cedar, Docker and Claude Code and get my hands dirty with a prod deploy on a barebones cloud instance. One meta learning was to be atomic with each commit and make them small, so I am not reverting a big chunk back each time I saw a bug. Software development best practices from 2021 are still relevant today.
