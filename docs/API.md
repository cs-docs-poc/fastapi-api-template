# API reference

This is an endpoint-by-endpoint reference for the Fastapi Template API. It's generated from, and should be read alongside, the OpenAPI spec at [`docs/openapi.yaml`](./openapi.yaml). For a description of the overall application, see [`docs/OVERVIEW.md`](./OVERVIEW.md).

Base URL in local development: `http://127.0.0.1:8000`

## Authentication

Every endpoint except `/auth/register`, `/auth/login` and `/health` requires a bearer token, obtained from `/auth/login` and sent on every request as:

```
Authorization: Bearer <access_token>
```

An access token is valid for 30 minutes; the accompanying refresh token is valid for 7 days. The template doesn't currently expose an endpoint to exchange a refresh token for a new access token, so a user has to log in again once their access token expires.

---

## Auth

### `POST /auth/register`

Registers a new user. Returns a 400 error if a user with the given email already exists.

**Request body**

```json
{
  "first_name": "ada",
  "last_name": "lovelace",
  "email": "ada@example.com",
  "password": "a-strong-password",
  "is_admin": false
}
```

**Response `200`**

```json
"ok"
```

**Response `400`**

```json
{ "detail": "User with this email already exist" }
```

### `POST /auth/login`

Exchanges an email and password for an access and refresh token. Sent as form data (`application/x-www-form-urlencoded`), following the OAuth2 password grant.

**Request body**

| Field | Required | Notes |
| --- | --- | --- |
| `username` | yes | the user's email address |
| `password` | yes | |
| `grant_type`, `scope`, `client_id`, `client_secret` | no | part of the OAuth2 password grant; unused by this template |

**Response `200`**

```json
{
  "access_token": "<jwt>",
  "refresh_token": "<jwt>"
}
```

**Response `400`**

```json
{ "detail": "Incorrect email or password" }
```

---

## Users

All endpoints in this section require a bearer token.

### `GET /users/get/users`

Returns every user, including each user's posts.

**Response `200`**

```json
[
  {
    "id": 1,
    "first_name": "ada",
    "last_name": "lovelace",
    "email": "ada@example.com",
    "is_admin": false,
    "creation_date": "2026-10-02",
    "posts": []
  }
]
```

**Response `404`**: returned when no users exist.

### `GET /users/get/user/{id}`

Returns a single user by ID, including their posts.

**Path parameters**: `id` (integer, required)

**Response `200`**: a single user object, shaped as above.

**Response `404`**: returned when no user with that ID exists.

### `DELETE /users/get/user/{id}`

Deletes a user by ID.

**Path parameters**: `id` (integer, required)

**Response `200`**

```json
"ok"
```

**Response `400`**: returned when the caller tries to delete their own account.

**Response `401`**: returned when the caller isn't an admin.

**Response `404`**: returned when no user with that ID exists.

---

## Posts

All endpoints in this section require a bearer token.

### `GET /posts/get/posts`

Returns every post.

**Response `200`**

```json
[
  {
    "id": 1,
    "title": "hello, world",
    "content": "first post",
    "user_id": 1,
    "creation_date": "2026-10-02"
  }
]
```

**Response `404`**: returned when no posts exist.

### `GET /posts/get/post/{id}`

Returns a single post by ID.

**Path parameters**: `id` (integer, required)

**Response `200`**: a single post object, shaped as above.

**Response `404`**: returned when no post with that ID exists.

### `POST /posts/add/post`

Creates a post for the given user ID.

**Request body**

```json
{
  "title": "hello, world",
  "content": "first post",
  "user_id": 1
}
```

**Response `200`**

```json
"ok"
```

The caller must be authenticated, but the endpoint doesn't currently check that `user_id` matches the caller's own ID.

### `DELETE /posts/delete/post/{id}`

Deletes a post by ID.

**Path parameters**: `id` (integer, required)

**Response `200`**

```json
"ok"
```

**Response `401`**: returned when the post's `user_id` doesn't match the caller's own ID.

**Response `404`**: returned when no post with that ID exists.

---

## Health

### `GET /health`

A basic liveness check. Doesn't require authentication.

**Response `200`**

```json
"ok"
```

---

## Validation errors

Any endpoint that takes a request body or a path parameter returns a 422 error on an invalid request, in the standard FastAPI shape:

```json
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error.email"
    }
  ]
}
```
