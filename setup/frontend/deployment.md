# Frontend Deployment

The Arukin frontend is a static SPA deployed to **Firebase Hosting**. All dynamic/secure logic runs in Supabase Edge Functions — the frontend is a static bundle with no server-side rendering.

## Build

```bash
cd frontend
npm run build
```

Vite outputs optimized static assets to `frontend/dist/`.

## Firebase Configuration

**.firebaserc** — links to the `arukinsec` project.

**firebase.json** — serves `dist/` as the public directory with a catch-all rewrite to `index.html` for SPA routing.

## Deploy

### Manual
```bash
firebase login
npm run build
firebase deploy --only hosting
```

### CI/CD
No automated CI/CD pipeline is currently configured. Manual deployment is the standard workflow. A future improvement would add a GitHub Actions workflow (`.github/workflows/firebase-hosting-merge.yml`) triggered on pushes to the default branch.

## Design Rationale

Static SPA deployment eliminates an entire class of server-side vulnerabilities on the frontend layer. There is no Node.js process, no server-side rendering, and no runtime environment on the hosting tier — just a CDN serving pre-built files.
