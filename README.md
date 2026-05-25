# this commit's issues

currently im getting 444s even when trying to hit valid `/ts2021`. this was after i implemented the changes on this commit that were parsed correctly by `nginx -t` on my vps

looking to at least allow `Go-http-client/2.0` ua from pfsense requests with a map

i was checking this page to [configure and harden nginx](https://wisdom.gitbook.io/gyan/nginx/core-concepts-and-basics) trying to figure out whats wrong with the current config; sort of matches with what llms complain about the config

### llm yapping

Because your security.conf is executed before location processing completes, and your $deny_ua map is matching the Tailscale/Headscale client.

This line is the culprit:

"~* (bot|crawler|scanner|curl|wget|python|perl|ruby|java|php)" 1;

Your logs show:

ua="Go-http-client/1.1"

That request is probably getting caught indirectly because of your if-based global filtering logic. The bigger issue is:

if ($deny_ua) { return 444; }

inside server {} runs during rewrite phase for all requests, including /ts2021.

Your allowed location does not bypass those checks.

What is happening

Request flow:

```
POST /ts2021
↓
server block entered
↓
include security.conf
↓
if ($deny_ua) return 444
↓
request terminated
↓
location never matters
```

So even though this exists:

`location ~ ^/(key|control|grpc|register|machine|ts2021|health|node)`

the request already died before proxying.

Another issue: your method filter

You only allow:

GET POST HEAD

Headscale/Tailscale uses more methods than that depending on version and gRPC transport:

POST
GET
OPTIONS
sometimes CONNECT/PUT in proxied setups

So your method map is also risky.

Best fix

Do NOT globally apply those filters to your Headscale routes.

Instead:

move security checks into the default-deny location
or exempt your API routes

Example:

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name hs.hapless.xyz;

    ssl_certificate "/etc/nginx/ssl/hs/fullchain.cer";
    ssl_certificate_key "/etc/nginx/ssl/hs/hs.hapless.xyz.key";

    ssl_protocols TLSv1.2 TLSv1.3;

    server_tokens off;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    # allowed headscale routes
    location ~ ^/(key|control|grpc|register|machine|ts2021|health|node) {
        proxy_pass http://127.0.0.1:8080;

        include /etc/nginx/http.d/hs/proxy.conf;
    }

    # everything else
    location / {

        if ($deny_request) { return 444; }
        if ($deny_method) { return 444; }
        if ($deny_ua) { return 444; }

        return 444;
    }
}

This way:

valid Headscale routes bypass the filters
scanners still get nuked
Also: your UA regex is too aggressive

Blocking:

(bot|crawler|scanner|curl|wget|python|perl|ruby|java|php)

is dangerous because:

health checks
APIs
monitoring
ACME clients
Go clients
automation

often use generic UAs.

I would reduce it to known malicious scanners only:

```
map $http_user_agent $deny_ua {
    default 0;
    "~* (nikto|sqlmap|nmap|nessus|openvas)" 1;
}
```

Another improvement

This:

`location ~ ^/(key|control|grpc|register|machine|ts2021|health|node)`

matches:

```
/ts2021evil
/nodewhatever
```

Better:

`location ~ ^/(key|control|grpc|register|machine|ts2021|health|node)(/|$)`

One more thing

You included:

`include /etc/nginx/conf.d/deny.conf;`

inside the vhost file but outside the server block.

That is valid for map, but usually cleaner in nginx.conf under http {}:

```
http {
    include conf.d/deny.conf;
}
```

since map belongs to HTTP context, not server context.

Recommended final setup
maps in http {}
no if in server root
whitelist valid locations
deny everything else
only block known bad scanners

That architecture is much safer and cleaner for Headscale.

but then how could i apply these security policies as a whole, cant i create a map that gets globally parsed so i dont have to put the ifs on each location / service i'll set up?

Yes — and that is the correct way to do it in nginx.

What you want is:

define reusable maps globally in http {}
compute ONE boolean like $block_request
apply it once centrally
selectively disable it per-location/service

The mistake is using if (...) return 444 at server level, because rewrite-phase execution happens before your location logic becomes useful.

Proper architecture
1. Global maps in http {}

Move all maps into something globally included:

http {
    include /etc/nginx/conf.d/deny.conf;
}
2. Build ONE final decision variable

Example:

map $uri $deny_request {
    default 0;

    "~* ^/(vendor|cgi-bin|phpunit|\.env|wp-admin|wp-login|HNAP1|boaform|xmlrpc\.php)" 1;
}

map $request_method $deny_method {
    default 1;

    GET  0;
    POST 0;
    HEAD 0;
    OPTIONS 0;
}

map $http_user_agent $deny_ua {
    default 0;

    "~* (nikto|sqlmap|nessus|openvas|nmap)" 1;
}

Then combine them:

map "$deny_request$deny_method$deny_ua" $block_request {
    default 0;

    "~1" 1;
}

Meaning:

if ANY component contains 1
request becomes blocked
3. Apply globally once

Now you can do:

server {
    listen 443 ssl http2;
    server_name example.com;

    if ($block_request) {
        return 444;
    }

    ...
}

This becomes reusable everywhere.

But how do you exempt routes like /ts2021?

Create another map.

4. Whitelist internal API routes

Example:

map $uri $security_bypass {
    default 0;

    "~^/(key|control|grpc|register|machine|ts2021|health|node)(/|$)" 1;
}

Then combine properly:

map "$security_bypass:$deny_request:$deny_method:$deny_ua" $block_request {
    default 0;

    # bypass enabled
    "~^1:" 0;

    # otherwise if any deny flag set
    "~:1" 1;
}

Now:

/ts2021
↓
security_bypass=1
↓
block_request=0
↓
allowed

while scanners still die.

This is the scalable nginx pattern

You can now reuse across all services:

headscale
grafana
gitea
jellyfin
nextcloud
APIs
static sites

without copy-pasting security logic.

Even cleaner approach: use geo

For IP filtering:

geo $bad_ip {
    default 0;

    1.2.3.4 1;
}

Then merge into block logic:

map "$bad_ip:$deny_ua:$deny_request" $block_request {

This scales very well.

Important nginx caveat

Using:

if ($block_request) {
    return 444;
}

is safe.

The infamous “if is evil” warning mainly applies to:

rewrites
nested location logic
proxy manipulation

Simple return is fine.

Recommended production layout
/etc/nginx/
├── nginx.conf
├── conf.d/
│   ├── maps.conf
│   ├── security.conf
│   └── headers.conf
├── sites/
│   ├── hs.conf
│   ├── grafana.conf
│   └── gitea.conf

Where:

maps.conf
→ all maps/geo definitions

security.conf
→ one reusable:

if ($block_request) { return 444; }
services only:
proxying
SSL
upstreams
app-specific logic

That keeps everything centralized and maintainable.



