# Social AI

Social AI is an image and video sharing web application with built-in AI image generation. Users sign up, describe an idea in plain words, generate an image with OpenAI, and publish it to a shared collection. They can also upload their own images and videos, and search posts by user or keyword.

The frontend is built with React. The backend is a Go service deployed on Google App Engine. Post data is stored in Elasticsearch running on a Google Compute Engine VM, and media files are stored in Google Cloud Storage.

## Features

- Sign up and sign in with JWT authentication
- Protected routes: the Create and Collection pages redirect to login when there is no token
- AI image generation from a text prompt with OpenAI `gpt-image-2`
- Lightbox preview with zoom, fullscreen and slideshow, and one-click upload of generated images
- Image and video upload with a description
- Collection page with separate Images and Videos tabs
- Search all posts, posts by a user, or posts whose description matches keywords
- Media files stored in Google Cloud Storage, post metadata stored in Elasticsearch, linked by the same UUID

## Tech Stack

| Area | Technology |
| --- | --- |
| Web UI | React 18, React Router 6, Axios, Ant Design, MUI, styled-components |
| Image display | react-photo-album, yet-another-react-lightbox |
| AI | OpenAI Node SDK (`gpt-image-2`) |
| Backend | Go, gorilla/mux, go-jwt-middleware, jwt-go |
| Search and storage | Elasticsearch 7 (olivere/elastic), Google Cloud Storage |
| Deployment | Google App Engine (flexible environment), Google Compute Engine |

## Architecture

```
Browser (React)
  ├── OpenAI API              prompt → base64 image
  └── Go service (App Engine) JSON / multipart requests with a Bearer token
        ├── Elasticsearch (GCE VM)   user and post documents
        └── Cloud Storage bucket     image and video files
```

When a post is uploaded, the backend generates a UUID for it. The file is saved to Cloud Storage with the UUID as the object name, and the post document (`id`, `user`, `message`, `url`, `type`) is saved to Elasticsearch with the same UUID as the document ID. The object's public link is stored in `url`, so the frontend can display the media directly.

## Project Structure

| Path | Responsibility |
| --- | --- |
| `src/components/App.js` | Reads the token from local storage and holds the login state. |
| `src/components/Main.js` | Defines the routes and redirects based on the login state. |
| `src/components/ResponsiveAppBar.js` | Navigation bar with Create, Collection, and logout. |
| `src/components/Login.js`, `Register.js` | Sign-in and sign-up forms. |
| `src/components/Landing.js` | The Create page: generates an image from a prompt, previews it, and uploads it. |
| `src/components/Collection.js` | Loads posts through search and shows them in Images and Videos tabs. |
| `src/components/SearchBar.js` | Search by all posts, keyword, or user. |
| `src/components/PhotoGallery.js` | Image grid and lightbox viewer. |
| `src/components/CreatePostButton.js`, `PostForm.js` | Upload dialog for local images and videos. |
| `server/main.go` | Loads the configuration, initializes Elasticsearch and Cloud Storage, and starts the server on port 8080. |
| `server/handler/` | HTTP routes, JWT middleware, CORS, and request handling. |
| `server/service/` | Sign-in, sign-up, upload, and search logic. |
| `server/backend/` | Elasticsearch and Cloud Storage clients. |
| `server/model/` | `User` and `Post` models. |
| `server/conf/deploy.example.yml` | Configuration template for the backend. |

## API

| Method | Endpoint | Auth | Description |
| --- | --- | --- | --- |
| POST | `/signup` | No | Create a user (`username`, `password`) |
| POST | `/signin` | No | Return a JWT that expires in 24 hours |
| POST | `/upload` | Bearer token | Upload a post as multipart form data (`message`, `media_file`) |
| GET | `/search` | Bearer token | Return all posts |
| GET | `/search?user=...` | Bearer token | Return posts from one user |
| GET | `/search?keywords=...` | Bearer token | Return posts whose message matches all keywords |

The media type is detected from the file extension: `.jpg`, `.jpeg`, `.png`, and `.gif` are images; `.mp4`, `.mov`, `.avi`, `.flv`, and `.wmv` are videos.

## Run Locally

### Frontend

Requirements:

- Node.js 18 or later
- An OpenAI API key
- A running backend

Install dependencies:

```bash
npm install
```

Create `.env` in the project root (see `.env.example`):

```
REACT_APP_OPENAI_KEY=your_openai_api_key
REACT_APP_API_BASE_URL=https://your-backend-url
```

Start the development server:

```bash
npm start
```

Open http://localhost:3000.

### Backend

Requirements:

- Go (see `server/go.mod`)
- An Elasticsearch 7 instance with basic authentication
- A Cloud Storage bucket, and Google Cloud credentials that can write to it

Copy the configuration template and fill in your own values:

```bash
cd server
cp conf/deploy.example.yml conf/deploy.yml
```

```yaml
elasticsearch:
  address: "http://<ES_HOST>:9200"
  username: "<ES_USERNAME>"
  password: "<ES_PASSWORD>"

gcs:
  bucket: "<GCS_BUCKET_NAME>"

token:
  secret: "<JWT_SIGNING_SECRET>"
```

Run the server:

```bash
go run .
```

On startup, the server creates the `user` and `post` indexes in Elasticsearch if they do not exist.

## Deployment

The backend is deployed to the App Engine flexible environment with `server/app.yaml`:

```bash
cd server
gcloud app deploy
```

Elasticsearch runs on a Compute Engine VM as a systemd service. The App Engine service joins the `default` VPC network, so it reaches Elasticsearch through the VM's internal IP address.

## Configuration

| Name | Where | Description |
| --- | --- | --- |
| `REACT_APP_OPENAI_KEY` | `.env` | OpenAI API key for image generation |
| `REACT_APP_API_BASE_URL` | `.env` | Backend URL |
| `elasticsearch.*` | `server/conf/deploy.yml` | Elasticsearch address and credentials |
| `gcs.bucket` | `server/conf/deploy.yml` | Cloud Storage bucket for media files |
| `token.secret` | `server/conf/deploy.yml` | Secret used to sign JWTs |

`.env` and `server/conf/deploy.yml` are ignored by Git.

## Current Scope

This is a learning project. Post deletion is not available yet: the frontend has delete buttons, but the delete route is disabled in the backend. The OpenAI API is called from the browser, so the API key is included in the frontend build; image generation should move to the backend before any public deployment. Passwords are stored without hashing, uploaded media is publicly readable, and there are no end-to-end tests.
