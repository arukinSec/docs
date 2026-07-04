# Frontend Deployment Guide

This document outlines the deployment strategy and processes for the ArukinSec frontend dashboard. We utilize **Firebase Hosting** for its global CDN, out-of-the-box SSL, and seamless Single Page Application (SPA) routing capabilities.

> [!NOTE]
> The deployment pipeline is largely automated via GitHub Actions, ensuring that all merges to the main branch are instantly reflected in production, and all Pull Requests are previewed in temporary staging environments.

## Build Process

Before deploying, the frontend must be compiled from React/TypeScript into optimized static assets (HTML, CSS, JS).

```bash
cd frontend
npm run build
```

This triggers the `vite build` command. Vite bundles the application and outputs the highly-optimized static files into the `dist/` directory.

### Design Rationale: Static SPA Deployment
By compiling the frontend into a static bundle without a Node.js server (e.g., Next.js SSR), we completely eliminate an entire class of server-side vulnerabilities on the frontend layer. The static files are served via Firebase CDN, and all dynamic/secure logic is strictly delegated to the Supabase Edge Functions.

## Firebase Hosting Configuration

Our Firebase setup is defined by two files:

### 1. `.firebaserc`
Links the local directory to the remote Firebase project (`arukinsec`).

```json
{
  "projects": {
    "default": "arukinsec"
  }
}
```

### 2. `firebase.json`
Configures how the hosting environment serves our files.

```json
{
  "hosting": {
    "public": "dist",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}
```

> [!IMPORTANT]
> The `rewrites` rule is critical for React Router. It tells Firebase to route all incoming requests (e.g., `/dashboard/settings`) back to `/index.html`, allowing the client-side router to handle the view logic without returning a 404 error.

## CI/CD Pipeline (GitHub Actions)

We employ automated workflows located in `.github/workflows/`.

1. **`firebase-hosting-pull-request.yml`**
   - Triggers on opening or updating a Pull Request.
   - Builds the frontend (`npm run build`).
   - Deploys the `dist/` folder to a temporary, isolated Firebase preview channel.
   - Comments the preview URL on the GitHub PR, allowing QA and stakeholders to test changes securely before merging.

2. **`firebase-hosting-merge.yml`**
   - Triggers when a PR is merged into the default branch.
   - Builds the frontend.
   - Deploys the build directly to the live Firebase production channel.

## Manual Deployment

In emergency situations or when bypassing CI, you can deploy manually using the Firebase CLI:

```bash
# Ensure you are logged into Firebase
firebase login

# Build the project
npm run build

# Deploy only the hosting target
firebase deploy --only hosting
```
