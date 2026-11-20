# CS-020 — Research References and Evidence Register

## Supplied documents

1. Codorythm 250 Case Studies catalogue.
2. System Design Knowledge Library.

## Technical references

- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- HTTP Caching, RFC 9111: https://www.rfc-editor.org/rfc/rfc9111
- HTTP/1.1 Message Syntax and Routing, RFC 9112: https://www.rfc-editor.org/rfc/rfc9112
- HTTP/2, RFC 9113: https://www.rfc-editor.org/rfc/rfc9113
- HTTP/3, RFC 9114: https://www.rfc-editor.org/rfc/rfc9114
- CONNECT and proxy request target concepts in RFC 9110/9112.
- OWASP SSRF Prevention: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- HTTP Request Smuggling security reference: https://portswigger.net/web-security/request-smuggling
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446
- Squid documentation: https://www.squid-cache.org/Doc/
- Envoy proxy documentation: https://www.envoyproxy.io/docs/envoy/latest/
- HAProxy documentation: https://docs.haproxy.org/
- NGINX proxy module: https://nginx.org/en/docs/http/ngx_http_proxy_module.html

## Claims requiring verification

- HTTP/1.1/2/3 support and protocol downgrade behavior.
- CONNECT timeout, port, and response behavior.
- TLS interception certificate, trust, and re-encryption semantics.
- Connection pooling, authority, credentials, and protocol compatibility.
- Cache private/shared/authorization/Vary behavior.
- DNS resolver rebinding protection and pinned socket behavior.
- Provider load-balancer, Kubernetes, and deployment limits.
