# Defense Bypass Knowledge Base

## Binary Exploitation

| Mitigation | Bypass Technique | Requirements |
|-----------|-----------------|--------------|
| NX/DEP | ROP, ret2libc, JOP | Gadget chain, stack control |
| ASLR | Info leak, partial overwrite | Memory disclosure primitive |
| Stack Canary | Format string leak, __stack_chk_fail overwrite | Leak primitive or write-what-where |
| PIE | 12-bit static partial overwrite, info leak | Base address disclosure |
| Full RELRO | __malloc_hook, __free_hook, exit hooks | Heap control, libc leak |
| Shadow Stack | Signal handler corruption, JOP | Specific kernel/config |

## Web Application

| Defense | Bypass Technique |
|---------|-----------------|
| WAF (ModSecurity) | Encoding variation (URL/Unicode/Hex), HTTP Parameter Pollution, HTTP Request Smuggling |
| CSP (Content Security Policy) | JSONP endpoints, AngularJS CSTI, script gadgets, DOM clobbering |
| Input Validation | Unicode normalization (NFD/NFC), double URL-encoding, null-byte injection, parser differentials |
| Rate Limiting | IP rotation, header spoofing (X-Forwarded-For), request pacing |
| CSRF Tokens | XSS → token extraction, same-origin bypass, token fixation |

## Network

| Defense | Bypass Technique |
|---------|-----------------|
| Firewall | Protocol tunneling (DNS/ICMP), port knocking, fragmentation |
| IDS/IPS | Payload obfuscation, encryption, session splicing |
| NAC | MAC spoofing, 802.1X bypass, VLAN hopping |

## Authentication

| Defense | Bypass Technique |
|---------|-----------------|
| MFA/2FA | MFA fatigue, SIM swap, OAuth token theft, session hijacking |
| Password Policy | Credential stuffing (breach DB), password spraying, kerberoasting |
| JWT | Algorithm confusion (none/HS256), key leakage, expired token reuse |
