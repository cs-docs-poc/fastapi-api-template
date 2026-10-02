# Fastapi Template documentation

This document describes the basic functionality of the Fastapi Template, a starting point for building an Application Programming Interface (API) with FastAPI. It's aimed at developers who are setting the template up for the first time or extending it for a proof of concept.

For the full list of endpoints, request and response formats, and error codes, see the OpenAPI spec at [`docs/openapi.yaml`](./openapi.yaml), or the companion reference in [`docs/API.md`](./API.md).

## What the template provides

The template is a minimal FastAPI application with:

* JSON Web Token (JWT) based authentication, scoped per endpoint
* user registration, lookup and deletion
* a posts resource, where each post belongs to a user
* a SQLite database by default, managed through Alembic migrations
* a basic health check endpoint for monitoring
* an async request/response path throughout, and type hints on the public interfaces
* a unit test suite covering authentication, users and posts

It's deliberately front end-independent: no templating engine or static file serving is included, and that choice is left to the developer extending the template.

## Project structure

The application code lives under `app/`:

* `app/app.py` creates the FastAPI application, registers the three routers, and adds Cross-Origin Resource Sharing (CORS) middleware for local development
* `app/main.py` is the entry point used by the application server (Uvicorn)
* `app/routers/` holds the `auth`, `users` and `posts` route handlers
* `app/db/models.py` defines the `Users` and `Posts` database tables with SQLAlchemy's Object-Relational Mapper (ORM)
* `app/db/schemas/` defines the Pydantic request and response models
* `app/db/sessions.py` configures the async database engine and session
* `app/deps.py` validates the bearer token on protected routes and resolves the current user
* `app/utils.py` handles password hashing and JWT creation
* `alembic/` holds the database migration scripts
* `tests/` holds the unit test suite

## Getting started

### Install the dependencies

Clone the repository, create a virtual environment, and install the pinned dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

### Configure the environment

Create a `.env` file in the project root:

```env
DATABASE_URL = sqlite+aiosqlite:///sql_app.db
DATABASE_TEST_URL = sqlite+aiosqlite:///test.db
JWT_SECRET_KEY = secret
JWT_REFRESH_SECRET_KEY = secret
```

`JWT_SECRET_KEY` signs access tokens, and `JWT_REFRESH_SECRET_KEY` signs refresh tokens. Both fall back to the literal value `secret` if they're not set, so they must be replaced with a strong, unique value before the application is used outside of local development.

### Set up the database

Database migrations are handled through Alembic. Apply them to create the SQLite database and its tables:

```bash
alembic upgrade head
```

The generated migration files are checked into source control under `alembic/versions/`, and any new migration should be added the same way.

### Run the application

```bash
uvicorn app.main:app --reload
```

The API is then available at `http://127.0.0.1:8000`, with interactive documentation at `/docs` and the raw OpenAPI schema at `/openapi.json`.

### Run the tests

```bash
pytest
```

The test suite spins up a separate SQLite database (`DATABASE_TEST_URL`) and exercises the full register, login, create and delete flow for both users and posts.

## Authentication

Authentication is handled by the `auth` router and follows the OAuth2 password grant.

A new account is created by posting to `/auth/register` with a first name, last name, email, password and an `is_admin` flag. Registration fails with a 400 error if a user with that email already exists. The password is hashed with bcrypt before it's stored, and the hash is never returned in a response.

A session is started by posting to `/auth/login` as form data, with `username` set to the user's email address and `password` set to their password. This field is named `username` because the OAuth2 specification requires it, not because the template expects a separate username. A successful login returns an access token, valid for 30 minutes, and a refresh token, valid for 7 days.

Every protected endpoint expects the access token on the `Authorization` header, as `Bearer <access_token>`. The `get_current_user` dependency in `app/deps.py` decodes the token, checks that it hasn't expired, and loads the matching user record; an expired or invalid token is rejected with a 401 or 403 error.

## Users

The `users` router manages user records, and every endpoint on it requires a valid access token.

* `GET /users/get/users` returns every registered user, including their posts
* `GET /users/get/user/{id}` returns a single user by ID
* `DELETE /users/get/user/{id}` deletes a user by ID, but only when the caller is an admin and isn't trying to delete their own account

## Posts

The `posts` router manages posts, and every endpoint on it also requires a valid access token.

* `GET /posts/get/posts` returns every post
* `GET /posts/get/post/{id}` returns a single post by ID
* `POST /posts/add/post` creates a post for a given user ID
* `DELETE /posts/delete/post/{id}` deletes a post, but only when the post's `user_id` matches the caller's own ID

## Health check

`GET /health` returns the literal string `ok` and doesn't require authentication. It's intended for use by a load balancer or monitoring system to confirm that the service is running.

## Known limitations

A few gaps are called out directly in the code, and are worth knowing about before this template is extended into a production service:

* user deletion has no accompanying unit tests
* post deletion checks the post's `user_id` against the caller's own ID rather than the post's actual author, so a caller can only delete a post if their own ID happens to match it
* SQLite is suitable for local development and proof-of-concept work, but isn't recommended for production; a database such as PostgreSQL or MySQL is a better fit there

## Further reading

* [`docs/API.md`](./API.md): an endpoint-by-endpoint reference, including request and response examples
* [`docs/openapi.yaml`](./openapi.yaml): the OpenAPI 3.0 specification, importable into tools such as Swagger UI or Postman
