# Cloudflare Deployment Strategy

This guide details the single-worker architecture for managing `geniux.net` as the unified apex domain.

## 1. Architecture Overview
We use a single Cloudflare Worker to handle traffic for the unified apex domain `geniux.net`. Any legacy traffic from `glog.geniux.net` is automatically 301 redirected to `geniux.net`.

- **Root Domain (`geniux.net`)**: Acts as a smart load balancer across multiple hosting providers (Vercel, Netlify, QCloud).
- **Legacy Subdomain (`glog.geniux.net`)**: Permanent 301 redirect to `geniux.net`.

## 2. Worker Configuration
The file `cloudflare.worker.js` contains the logic.

### Domain Routing Logic:
```javascript
if (url.hostname === "glog.geniux.net") {
    let targetPath = url.pathname;
    if (targetPath === "/" || targetPath === "") {
        targetPath = "/blog";
    }
    return Response.redirect(`https://geniux.net${targetPath}${url.search}`, 301);
}
if (url.hostname === "geniux.net" || url.hostname === "www.geniux.net") {
    return await handleRequest(request);
}
```

## 3. Do you need two Workers?
**No.** A single worker is sufficient. Cloudflare allows you to map multiple routes to the same worker:
1. `geniux.net/*` -> `glog-worker`
2. `glog.geniux.net/*` -> `glog-worker`

## 4. Step-by-Step Deployment

### A. Deploy via Wrangler
1. Ensure `wrangler.toml` is configured (if not present, create one or use CLI).
2. Run:
   ```bash
   npx wrangler deploy cloudflare.worker.js --name glog-worker
   ```

### B. Configure Custom Domains in Cloudflare Dashboard
1. Go to **Workers & Pages** -> **glog-worker** -> **Triggers**.
2. Under **Custom Domains**, add:
   - `geniux.net`
   - `glog.geniux.net`
3. Cloudflare will automatically handle DNS and SSL.

### C. Set Environment Variables
In the Cloudflare Dashboard (or via wrangler), ensure any required environment variables for the worker are set (though currently the worker uses hardcoded backends).

## 5. Benefits of this Setup
- **Edge Performance**: HTML is cached at the edge for 10 minutes.
- **Failover**: If Vercel goes down, the worker automatically switches to Netlify or QCloud.
- **Unified Brand**: One worker manages your entire web presence.
