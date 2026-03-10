# Multi-Tenant Deployment Documentation
[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

Thanks for visiting my GitHub account!

## Next.js + Laravel + Wildcard SSL + PM2 + Nginx + Cloudflare


## 1. What is Multi-Tenant Architecture and How It Works

### Concept

Multi-tenant architecture is a system where a single application serves multiple
customers (tenants), each with their own isolated data and experience, but all
running on the same codebase and infrastructure.

In this project, each company/tenant gets their own subdomain:
```
rahat.aerohire.io       -> Rahat company's career page
techcorp.aerohire.io    -> TechCorp's career page
startupxyz.aerohire.io  -> StartupXYZ's career page
```

All of these subdomains point to the same server and the same Next.js application.
The application detects which tenant is requesting by reading the subdomain from
the Host header, then loads that tenant's specific data.

### How Tenant Detection Works (Step by Step)
```
1. User visits:       rahat.aerohire.io
2. Cloudflare DNS:    *.aerohire.io  ->  YOUR_SERVER_IP
3. Nginx receives:    request for rahat.aerohire.io
4. Nginx matches:     server_name *.aerohire.io  ->  proxy to port 3001
5. Next.js reads:     Host header = "rahat.aerohire.io"
6. Next.js extracts:  subdomain = "rahat"
7. Next.js queries:   database for tenant where slug = "rahat"
8. Next.js renders:   that tenant's career page with their jobs, branding, etc.
```

### Why Wildcard DNS + SSL is Essential for Multi-Tenant

Without wildcard setup, every time a new tenant is created you would need to:
- Manually add a new DNS record for their subdomain
- Issue a new SSL certificate for their subdomain
- Update Nginx config for their subdomain

With wildcard setup:
- `*.aerohire.io` DNS record covers ALL subdomains automatically
- `*.aerohire.io` SSL certificate covers ALL subdomains automatically
- Nginx wildcard `server_name *.aerohire.io` handles ALL subdomains automatically
- New tenants are created by just adding a record to the database — no server config changes needed

### Tenant Detection in Laravel (Backend)
```php
$host = $request->getHost();           // rahat.aerohire.io
$subdomain = explode('.', $host)[0];   // rahat
$tenant = Tenant::where('slug', $subdomain)->firstOrFail();
```

### Tenant Detection in Next.js (Frontend)
```javascript
// In getServerSideProps or middleware
const host = req.headers.host;                    // rahat.aerohire.io
const subdomain = host.split('.')[0];             // rahat
const tenant = await fetchTenant(subdomain);      // fetch from API or DB
```

Nginx must forward the Host header correctly for this to work:
```nginx
proxy_set_header Host $host;
```

---

## 2. Project Structure
```
/home/aerohire/htdocs/aerohire.io
├── aerohire-main/               # Laravel backend + main landing site
│   ├── app/
│   ├── bootstrap/
│   ├── config/
│   ├── public/
│   ├── routes/
│   ├── storage/
│   └── artisan
│
└── aerohire-frontend/           # Next.js frontend (tenant career pages)
    ├── pages/
    ├── public/
    ├── components/
    ├── package.json
    ├── next.config.js
    └── node_modules/
```

- `aerohire-main` runs on port `3000` — serves main site and Laravel API
- `aerohire-frontend` runs on port `3001` — serves all tenant subdomain pages

---

## 3. How the System Works (Full Flow)
```
Browser Request
      |
      v
Cloudflare (DNS + Proxy)
      |
      v
Nginx (Reverse Proxy on Server)
      |
      |-- aerohire.io / www.aerohire.io  -->  127.0.0.1:3000  (Laravel)
      |
      |-- *.aerohire.io                  -->  127.0.0.1:3001  (Next.js)
                                                    |
                                                    v
                                          Next.js reads Host header
                                          extracts subdomain
                                          loads tenant-specific data
```

---

## 4. Running the Projects

### Laravel (Main Site — Port 3000)
```bash
cd /home/aerohire/htdocs/aerohire.io/aerohire-main

composer install
cp .env.example .env
php artisan key:generate
php artisan migrate --seed

pm2 start "php artisan serve --host=127.0.0.1 --port=3000" --name aerohire-main
```

### Next.js (Tenant Frontend — Port 3001)
```bash
cd /home/aerohire/htdocs/aerohire.io/aerohire-frontend

npm install
npm run build

pm2 start "npm start" --name aerohire-frontend
```

### PM2 Process Management
```bash
pm2 list                          # See all running processes
pm2 logs aerohire-main            # View Laravel logs
pm2 logs aerohire-frontend        # View Next.js logs
pm2 restart aerohire-main         # Restart Laravel
pm2 restart aerohire-frontend     # Restart Next.js
pm2 stop aerohire-frontend        # Stop a process
pm2 delete aerohire-frontend      # Remove from PM2
pm2 save                          # Save process list
pm2 startup                       # Enable auto-start on server reboot
```

---

## 5. What is Wildcard SSL and How It Works

A standard SSL certificate covers only one specific domain such as `aerohire.io`.
A wildcard certificate covers the root domain and all first-level subdomains under
it using a single certificate.
```
*.aerohire.io  covers:
  - tenant1.aerohire.io
  - rahat.aerohire.io
  - company-xyz.aerohire.io
  - any-new-subdomain.aerohire.io
```

Without wildcard SSL, a new certificate would be required for every new tenant
subdomain, which is not feasible in a dynamic multi-tenant system.

### Why Cloudflare DNS is Required for Wildcard SSL

Let's Encrypt issues wildcard certificates only via the DNS-01 challenge. This
means Let's Encrypt verifies domain ownership by checking a temporary TXT record
in your DNS. Cloudflare's API allows Certbot to automatically create and remove
this TXT record during certificate issuance, making the process fully automated.

---

## 6. Cloudflare Setup

### Step 1 — Create a Cloudflare API Token

1. Go to https://dash.cloudflare.com
2. Click your profile icon at the top right
3. Select **My Profile**
4. Go to the **API Tokens** tab
5. Click **Create Token**
6. Select the template **Edit zone DNS**
7. Under **Zone Resources** set it to `Include > Specific zone > aerohire.io`
8. Click **Continue to summary**, then **Create Token**
9. Copy the token immediately — it will not be shown again

### Step 2 — Add DNS Records in Cloudflare

Go to your domain in Cloudflare Dashboard, then **DNS > Records**.

| Type | Name | Content         | Proxy Status |
|------|------|-----------------|--------------|
| A    | @    | YOUR_SERVER_IP  | Proxied      |
| A    | www  | YOUR_SERVER_IP  | Proxied      |
| A    | *    | YOUR_SERVER_IP  | Proxied      |

The `*` A record makes all subdomains resolve to your server automatically.
Proxied (orange cloud) means traffic passes through Cloudflare first.

### Step 3 — Cloudflare SSL/TLS Setting

In Cloudflare Dashboard go to **SSL/TLS > Overview** and set the mode to
**Full (strict)**.

---

## 7. Wildcard SSL Certificate Issuance with Certbot

### Install Required Packages
```bash
apt update
apt install certbot python3-certbot-dns-cloudflare -y
```

### Configure Cloudflare Credentials
```bash
mkdir -p ~/.secrets
nano ~/.secrets/cloudflare.ini
```

Add this line and replace with your actual token:
```
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE
```
```bash
chmod 600 ~/.secrets/cloudflare.ini
```

### Issue the Wildcard Certificate
```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
  -d aerohire.io \
  -d "*.aerohire.io" \
  --email your@email.com \
  --agree-tos
```

### Certificate File Locations
```
/etc/letsencrypt/live/aerohire.io-0001/fullchain.pem
/etc/letsencrypt/live/aerohire.io-0001/privkey.pem
```

Note: The `-0001` suffix appears when a certificate for that domain already existed
previously. Always verify the actual folder name with:
```bash
ls /etc/letsencrypt/live/
```

### Auto-Renewal Verification
```bash
systemctl status certbot.timer
certbot renew --dry-run
```

---

## 8. Nginx Configuration

File location: `/etc/nginx/sites-enabled/aerohire.io.conf`
```nginx
# Redirect all HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name aerohire.io *.aerohire.io;
    return 301 https://$host$request_uri;
}

# MAIN SITE (aerohire.io)
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name aerohire.io www.aerohire.io;
    root /home/aerohire/htdocs/aerohire.io/aerohire-main;

    ssl_certificate     /etc/letsencrypt/live/aerohire.io-0001/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aerohire.io-0001/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    index index.html;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# TENANT SUBDOMAINS (Next.js career pages)
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name *.aerohire.io;
    root /home/aerohire/htdocs/aerohire.io/aerohire-frontend;

    ssl_certificate     /etc/letsencrypt/live/aerohire.io-0001/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aerohire.io-0001/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    index index.html;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Apply Config Changes
```bash
nginx -t
systemctl reload nginx
```

---

## 9. Debugging and Troubleshooting

### Nginx
```bash
nginx -t
systemctl reload nginx
systemctl status nginx
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log
```

### SSL Certificate
```bash
# Check domains covered by the certificate
openssl x509 -in /etc/letsencrypt/live/aerohire.io-0001/fullchain.pem \
  -text -noout | grep -A2 "Subject Alternative Name"

# Check expiry date
openssl x509 -in /etc/letsencrypt/live/aerohire.io-0001/fullchain.pem \
  -text -noout | grep -E "Not Before|Not After"

# Test SSL connection to a subdomain
openssl s_client -connect rahat.aerohire.io:443 -servername rahat.aerohire.io

# Test via curl
curl -vI https://rahat.aerohire.io 2>&1 | grep -E "SSL|issuer|HTTP"
```

### PM2
```bash
pm2 list
pm2 logs aerohire-main --lines 50
pm2 logs aerohire-frontend --lines 50
```

---

## 10. Common Issues and Fixes

| Issue | Cause | Fix |
|-------|-------|-----|
| `SSL certificate problem: unable to get local issuer certificate` | Certificate chain incomplete | Use `fullchain.pem`, not just `cert.pem` |
| `SSL: no alternative certificate subject name matches target host name` | Wrong cert path, missing wildcard SAN | Point nginx to correct `aerohire.io-0001` path |
| `self-signed certificate in certificate chain` | Using Cloudflare Origin Cert without CF proxy | Switch to Let's Encrypt certificate |
| `DNS_PROBE_FINISHED_NXDOMAIN` | Wildcard `*` A record missing in Cloudflare | Add `* -> YOUR_SERVER_IP` A record |
| `502 Bad Gateway` | PM2 process not running or wrong port | Check `pm2 list`, verify ports 3000/3001 |
| `ERR_TOO_MANY_REDIRECTS` | Cloudflare SSL mode set to Flexible | Set Cloudflare SSL/TLS to Full (strict) |
| CloudPanel vhost showing old config | CloudPanel stores its own copy | Update vhost in CloudPanel dashboard UI |
| `certbot renew` fails | Cloudflare token expired | Re-create API token, update `cloudflare.ini` |
| Tenant page not loading correct data | Host header not forwarded | Ensure `proxy_set_header Host $host` is set in nginx |
| New subdomain not resolving | Wildcard DNS not propagated yet | Wait for DNS propagation or check Cloudflare records |

---

## 11. Summary Reference

| Component | Location / Value |
|-----------|-----------------|
| Laravel main site | `/home/aerohire/htdocs/aerohire.io/aerohire-main` |
| Next.js frontend | `/home/aerohire/htdocs/aerohire.io/aerohire-frontend` |
| Laravel PM2 process | `aerohire-main` on port `3000` |
| Next.js PM2 process | `aerohire-frontend` on port `3001` |
| Nginx config file | `/etc/nginx/sites-enabled/aerohire.io.conf` |
| SSL certificate | `/etc/letsencrypt/live/aerohire.io-0001/fullchain.pem` |
| SSL private key | `/etc/letsencrypt/live/aerohire.io-0001/privkey.pem` |
| Cloudflare credentials | `~/.secrets/cloudflare.ini` |

Documentation এ Backend Configuration section add করুন। পুরো document এর পরে এই section টা add করুন:

---

## 12. Backend Configuration (Laravel Multi-Tenant)

### How the Backend Identifies Tenants

Every API request to the career page routes passes through the `IdentifyTenant` middleware. The middleware reads the incoming request's host and resolves which tenant is making the request before the controller ever runs.

The resolution follows this priority order:

```
1. Check X-Tenant-Domain header (for custom domain support)
2. Check career_page_custom_domain column in tenants table
3. Fall back to subdomain matching against tenant slug
4. If no tenant found, return 403 error
```

### Middleware — IdentifyTenant

File: `app/Http/Middleware/IdentifyTenant.php`

```php
<?php
namespace App\Http\Middleware;

use App\Models\Tenant;
use App\Traits\ApiResponse;
use Closure;
use Illuminate\Http\Exceptions\HttpResponseException;

class IdentifyTenant
{
    use ApiResponse;

    public function handle($request, Closure $next)
    {
        $host = $request->header('X-Tenant-Domain') ?? $request->getHost();

        // Priority 1: Check if host matches a custom domain
        $tenant     = Tenant::where('career_page_custom_domain', $host)->first();
        $baseDomain = config('tenant.base_domain');

        // Priority 2: Check if host is a subdomain of base domain
        if (! $tenant && str_ends_with($host, $baseDomain)) {
            $subdomain = explode('.', $host)[0];
            $tenant    = Tenant::where('slug', $subdomain)->first();
        }

        if (! $tenant) {
            throw new HttpResponseException(
                $this->error(null, 'Tenant not found', 403)
            );
        }

        // Bind resolved tenant to the service container
        app()->instance('tenant', $tenant);

        return $next($request);
    }
}
```

### How Tenant Resolution Works Step by Step

```
Request arrives at:  rahat.aerohire.io/api/career-page
                              |
                              v
Middleware reads Host header: "rahat.aerohire.io"
                              |
                              v
Check X-Tenant-Domain header? Not present
                              |
                              v
Check career_page_custom_domain = "rahat.aerohire.io"? Not found
                              |
                              v
Does host end with ".aerohire.io"? Yes
                              |
                              v
Extract subdomain: "rahat"
                              |
                              v
Query: Tenant::where('slug', 'rahat')->first()
                              |
                              v
Tenant found -> bind to app('tenant')
                              |
                              v
Request continues to Controller
```

### Environment Configuration

In `.env` file, set the base domain so the middleware knows what to strip:

```env
TENANT_BASE_DOMAIN=aerohire.io
```

In `config/tenant.php`:

```php
return [
    'base_domain' => env('TENANT_BASE_DOMAIN', 'aerohire.io'),
];
```

### Routes

File: `routes/api.php`

```php
Route::middleware('identify.tenant')->group(function () {
    Route::get('/career-page', [PublicJobController::class, 'careerPage']);
    Route::get('/career-page/{id}', [PublicJobController::class, 'show']);
});
```

Register the middleware in `app/Http/Kernel.php`:

```php
protected $routeMiddleware = [
    // ...
    'identify.tenant' => \App\Http\Middleware\IdentifyTenant::class,
];
```

### Controller — Career Page

File: `app/Http/Controllers/PublicJobController.php`

```php
public function careerPage(Request $request)
{
    $request->merge([
        'status' => $request->input('status', 'online'),
    ]);

    // Retrieve tenant that was resolved and bound by the middleware
    $tenant = app('tenant');
    $tenant->load(['benefits', 'offices:id,label', 'media']);

    $jobs             = $this->jobService->getTenantJobs($tenant, $request->all());
    $filteredContents = $this->jobService->getJobFilteredContents($tenant);

    // Determine redirect URL: custom domain takes priority over subdomain
    $redirect_url = ($tenant->is_permission_custom_domain && $tenant->career_page_custom_domain)
        ? $tenant->career_page_custom_domain
        : $tenant->slug . '.' . config('tenant.base_domain');

    return $this->success([
        'filters_data' => $filteredContents,
        'company'      => [
            'id'              => $tenant->id,
            'name'            => $tenant->name,
            'slug'            => $tenant->slug,
            'redirect_url'    => $redirect_url,
            'company_type'    => $tenant->company_type,
            'employeesNumber' => $tenant->employeesNumber
                                    ? new EmployeeNumberResource($tenant->employeesNumber)
                                    : null,
            'website'         => $tenant->website,
            'address'         => $tenant->address,
            'logo'            => $tenant->logo_path,
            'banner'          => $tenant->career_page_banner_path,
            'about'           => $tenant->about,
            'mission'         => $tenant->mission,
            'benefits'        => BenefitResource::collection($tenant->benefits),
            'media'           => MediaResource::collection($tenant->media),
            'social_links'    => SocialLinkResource::collection($tenant->socialLinks),
        ],
        'jobs'         => CareerPageJobResource::collection($jobs)->withQueryString(),
    ], 'Company jobs fetched successfully', 200);
}
```

### Custom Domain Support

The middleware also supports tenants who have their own custom domain (e.g. `careers.rahatcompany.com` instead of `rahat.aerohire.io`).

For this to work:
- The tenant's `career_page_custom_domain` column must be set in the database
- The custom domain's DNS must point to your server IP
- Nginx must have a server block or the wildcard must catch it
- The `X-Tenant-Domain` header can also be sent manually from the frontend for cases where the host cannot be read directly

### Nginx Host Header — Why It Is Critical

In the Nginx config, this line ensures Laravel receives the original subdomain:

```nginx
proxy_set_header Host $host;
```

Without this, `$request->getHost()` in Laravel would return `127.0.0.1` instead of `rahat.aerohire.io`, and tenant resolution would fail every time.

### Backend Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| `Tenant not found 403` on valid subdomain | `proxy_set_header Host $host` missing in nginx | Add the header in nginx location block |
| `Tenant not found 403` always | Slug does not match subdomain | Check `tenants` table, verify `slug` column value |
| Custom domain not resolving to correct tenant | `career_page_custom_domain` not set in DB | Update tenant record in database |
| `config('tenant.base_domain')` returns null | `TENANT_BASE_DOMAIN` missing in `.env` | Add `TENANT_BASE_DOMAIN=aerohire.io` to `.env` and run `php artisan config:cache` |
| Works locally but not on server | APP_URL or trusted proxy not configured | Set `TRUSTED_PROXIES=*` in `.env` for servers behind Cloudflare |

---
## Follow Me

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='40'>](https://github.com/learnwithfair) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='40'>](https://www.facebook.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='40'>](https://www.instagram.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/twitter.svg' alt='twitter' height='40'>](https://www.twiter.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='40'>](https://www.youtube.com/@learnwithfair)

 <!-- MARKDOWN LINKS & IMAGES  -->

[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair
