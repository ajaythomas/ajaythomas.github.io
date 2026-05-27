---
title: Building with Claude Code: A FamilyOffice app written in FastAPI and React 
categories:
- Tech
feature_image: "https://picsum.photos/2560/600?image=872"
published: false
---

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

### Logging


### Cedar Authorization


## VSCode IDE and plugins

1. Claude Code
    1. Note that memory is separate and isolated between Claude Code interfaces (i.e. your conversation on VSCode Claude Code vs Claude Mac app vs Claude Code on your Mac terminal)
1. autoDocstring - python docstring generator
1. Cedar (for my authz policies)
1. Code Spell Checker
1. Microsoft's tools like Container Tools, MyPy Type Checker, PostgresSQL (which I used to connect to both localhost and Hetzner cloud postgres db docker containers)
1. Ruff

## Docker containers and SSH

https://docs.google.com/document/d/1a8ekIqG8aFFRYOUVkr8jhbk3opdssakfvs9Lw5MmP0c/edit?tab=t.fyv9kr8vxfco 
Docker profiles https://github.com/ajaythomas/family_office/blob/main/docker-compose.yml 

## OAuth: Google Calendar 


## Production Deployment

Hetzner Cloud
Caddy file
Env secrets 


## Claude Code

Claude Plan https://github.com/ajaythomas/family_office/blob/main/plan.md

Claude md https://github.com/ajaythomas/family_office/blob/main/CLAUDE.md 


## FrontEnd

