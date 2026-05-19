# Deployment Guide: ipt-2026-frontend to Render

This document summarizes the changes made to **ipt-2026-frontend** to prepare it for deployment as a Render Static Site, and outlines the correct configuration required for both the frontend and backend environments to communicate securely in production.

---

## 1. Frontend Configuration Changes

### A. Disable Fake Backend in Production
The fake backend interceptor has been successfully disabled in production environments. This ensures that live API calls targeting `/accounts/*` reach the real deployed backend instead of being intercepted.

* **Source File:** `src/app/app.module.ts`
* **Implementation:**
  ```typescript
  providers: [
    { provide: APP_INITIALIZER, useFactory: appInitializer, multi: true, deps: [AccountService] },
    { provide: HTTP_INTERCEPTORS, useClass: JwtInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },

    // Disable fake backend provider in production
    ...(environment.production ? [] : [fakeBackendProvider])
  ],
  ```

### B. Production Environment API Configuration
The production environment has been configured to target the live backend API.

* **Source File:** `src/environments/environment.prod.ts`
* **Implementation:**
  ```typescript
  export const environment = {
    production: true,
    apiUrl: 'https://ipt-2026-backend.onrender.com'
  };
  ```

### C. Build Budgets Adjusted
To guarantee a clean build in production pipelines without any warnings or failures, the initial bundle budget has been adjusted to accommodate the final production size.

* **Source File:** `angular.json`
* **Change:** Raised `maximumWarning` budget for initial bundle from `500kb` to `600kb`.

---

## 2. Render Static Site Configuration (Frontend)

To deploy the Angular application as a **Render Static Site**, use the following configuration in your Render Dashboard:

| Parameter | Configuration Value |
| :--- | :--- |
| **Service Type** | Static Site |
| **Branch** | `main` |
| **Build Command** | `npm ci && npm run build` |
| **Publish Directory** | `dist/ipt-2026-frontend` |

### ⚠️ Critical SPA Routing Rule (Redirects/Rewrites)
Single Page Applications (SPAs) require all route requests to be handled by `index.html` so that Angular Router can handle deep linking. 
* **Do NOT use a Redirect rule** (this returns a `301`/`302` which strips out route query parameters and breaks pages like email verification).
* **DO use a Rewrite rule**.

Configure your Render Static Site under **Redirects/Rewrites** with the following:

| Source | Destination | Action |
| :--- | :--- | :--- |
| `/*` | `/index.html` | **Rewrite** |

*This setup ensures that deep links (e.g., `/account/verify-email?token=...` or `/account/login`) load correctly and retain their query parameters.*

---

## 3. Mandatory Backend Environment Variables

Since the frontend communicates with the backend using refresh-token cookies (`withCredentials: true`), the backend must explicitly allow credentials and configure cookies properly for **cross-site** requests.

> [!WARNING]
> Because both frontend and backend are hosted on Render subdomains (e.g. `*.onrender.com`), browsers treat them as completely **cross-site** because `onrender.com` is on the Public Suffix List. 
> 
> Therefore, using `COOKIE_SAMESITE=lax` **will fail** to send refresh-token cookies during background AJAX requests. You **must** use `SameSite=None` with `Secure=true`.

Set the following environment variables on your deployed backend:

```ini
# Deployed frontend origin (no trailing slash)
CORS_ORIGIN=https://<your-render-frontend-domain>.onrender.com

# Must be set to true for HTTPS secure cookie transmission
COOKIE_SECURE=true

# MUST be set to "none" for cross-site cookie transmission
COOKIE_SAMESITE=none
```

---

## 4. Post-Deployment Verification Checklist

1. **Verify Deep Links:**
   Refresh the page when on a deep route like `/account/login` to ensure the Render Static Site serves `index.html` via the Rewrite rule, and Angular loads the page without a 404.
2. **Verify Email Link Redirect:**
   Click an email verification link (e.g., `/account/verify-email?token=...`). The browser URL must stay exactly as requested, and the console/network tab should show a `POST` request to `https://ipt-2026-backend.onrender.com/accounts/verify-email` returning a `200 OK`.
3. **Verify CORS & Cookies:**
   Log in to the app and confirm that subsequent requests (or page refreshes) correctly send the cookies and execute the `/accounts/refresh-token` handshake.
