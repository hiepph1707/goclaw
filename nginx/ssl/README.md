# SSL Certificates

Place your SSL certificate files here:

- `fullchain.pem` — Full certificate chain (server cert + intermediate CAs)
- `private.key` — Private key (keep secure, never commit to git)

## Generate self-signed cert for development

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout private.key -out fullchain.pem \
  -subj "/CN=localhost"
```

## Important

These files are listed in `.gitignore` and should NOT be committed to version control.
