# Django on Vercel with JWT Authentication

A Django REST API deployed on Vercel's serverless platform with JWT authentication and PostgreSQL via Neon.

## API Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/api/auth/register/` | Create account, returns JWT tokens | No |
| POST | `/api/auth/login/` | Email + password, returns tokens | No |
| POST | `/api/auth/logout/` | Blacklist refresh token | Yes |
| POST | `/api/auth/token/refresh/` | Get new access token | No |
| POST | `/api/auth/token/verify/` | Verify token validity | No |
| GET/PATCH | `/api/auth/me/` | Get/update current user | Yes |

## Setup

### 1. Clone and install dependencies

```bash
git clone <repo-url>
cd vercel-django-spike
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```
DATABASE_URL=postgres://user:password@host:5432/dbname
SECRET_KEY=your-secret-key-here
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

Generate a secret key:

```bash
python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

For local development without Postgres, you can leave `DATABASE_URL` blank — it falls back to SQLite.

### 3. Run migrations

```bash
python3 manage.py migrate
```

### 4. Start the dev server

```bash
python3 manage.py runserver
```

The API is now available at `http://127.0.0.1:8000/api/auth/`.

## Deploy to Vercel

### 1. Install the Vercel CLI

```bash
npm i -g vercel
```

### 2. Create a Neon database

Sign up at [neon.tech](https://neon.tech) and create a new project. Copy the connection string — it looks like:

```
postgres://username:password@ep-something.region.aws.neon.tech/dbname?sslmode=require
```

### 3. Link and deploy

From the project directory:

```bash
vercel
```

Follow the prompts to link to your Vercel account and project.

### 4. Set environment variables

```bash
echo '<your-secret-key>' | vercel env add SECRET_KEY production
echo '<your-neon-connection-string>' | vercel env add DATABASE_URL production
echo '<your-domain>.vercel.app' | vercel env add ALLOWED_HOSTS production
echo 'https://<your-frontend-origin>' | vercel env add CORS_ALLOWED_ORIGINS production
```

For `ALLOWED_HOSTS`, use your Vercel domain (e.g. `my-project.vercel.app`). You can add multiple comma-separated values.

For `CORS_ALLOWED_ORIGINS`, add every origin that will consume the API, comma-separated.

### 5. Deploy to production

```bash
vercel --prod
```

### 6. Run migrations against production

Migrations cannot run from serverless functions. Run them locally against your Neon database by setting `DATABASE_URL` in your `.env` to the Neon connection string, then:

```bash
python3 manage.py migrate
```

## Using the API

Replace `BASE_URL` with `http://127.0.0.1:8000` for local development or your Vercel deployment URL (e.g. `https://my-project.vercel.app`).

All requests use `Content-Type: application/json`.

### Register a new user

```
POST {BASE_URL}/api/auth/register/
```

Body:

```json
{
  "email": "user@example.com",
  "username": "myuser",
  "password": "securepass123",
  "password_confirm": "securepass123"
}
```

Response `201`:

```json
{
  "user": {
    "id": 1,
    "email": "user@example.com",
    "username": "myuser",
    "first_name": "",
    "last_name": ""
  },
  "tokens": {
    "refresh": "eyJ...",
    "access": "eyJ..."
  }
}
```

Save both tokens. The `access` token is used to authenticate requests. The `refresh` token is used to get a new access token when it expires.

### Login

```
POST {BASE_URL}/api/auth/login/
```

Body:

```json
{
  "email": "user@example.com",
  "password": "securepass123"
}
```

Response `200`:

```json
{
  "refresh": "eyJ...",
  "access": "eyJ..."
}
```

### Get current user

```
GET {BASE_URL}/api/auth/me/
```

Header:

```
Authorization: Bearer <access_token>
```

Response `200`:

```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "myuser",
  "first_name": "",
  "last_name": ""
}
```

### Update current user

```
PATCH {BASE_URL}/api/auth/me/
```

Header:

```
Authorization: Bearer <access_token>
```

Body (include only the fields you want to update):

```json
{
  "first_name": "Jane",
  "last_name": "Doe"
}
```

Response `200`: returns the updated user object.

### Refresh access token

Access tokens expire after 15 minutes. Use the refresh token to get a new one.

```
POST {BASE_URL}/api/auth/token/refresh/
```

Body:

```json
{
  "refresh": "<refresh_token>"
}
```

Response `200`:

```json
{
  "access": "eyJ...",
  "refresh": "eyJ..."
}
```

Note: refresh token rotation is enabled — each refresh returns a new refresh token and blacklists the old one. Always store the new refresh token.

### Verify a token

```
POST {BASE_URL}/api/auth/token/verify/
```

Body:

```json
{
  "token": "<access_or_refresh_token>"
}
```

Response `200` with an empty body if valid. Response `401` if invalid or expired.

### Logout

```
POST {BASE_URL}/api/auth/logout/
```

Header:

```
Authorization: Bearer <access_token>
```

Body:

```json
{
  "refresh": "<refresh_token>"
}
```

Response `205` with no content. The refresh token is blacklisted and can no longer be used.

### Typical workflow in Postman/Hoppscotch

1. **Register** or **Login** to get your tokens
2. For authenticated requests, go to the **Authorization** tab (Postman) or **Authorization** section (Hoppscotch), select **Bearer Token**, and paste the `access` token
3. When you get a `401` response, call **Refresh** with your refresh token to get a new access token
4. On **Logout**, send the refresh token in the body to blacklist it

## Key things to know

- **`api/wsgi.py`** exposes both `application` (for Django's dev server) and `app` (for Vercel's runtime). Don't remove either.
- **`vercel.json`** routes all requests through `api/wsgi.py`. The `builds` config overrides any Build Settings in the Vercel dashboard — this is expected.
- **`CONN_MAX_AGE=0`** in the database config disables persistent connections, which is correct for serverless. Neon's built-in connection pooler handles this.
- **Token blacklisting** is enabled for logout support. Expired blacklisted tokens should be cleaned up periodically with `python3 manage.py flushexpiredtokens` (run manually or via external cron — there are no background workers in serverless).
- **`AUTH_USER_MODEL`** uses a custom User model with email as the login field. This must be set before running the first migration.
- **Read-only filesystem** at runtime — no file uploads without external storage (e.g. S3).
