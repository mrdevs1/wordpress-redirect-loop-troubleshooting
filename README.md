# WordPress Redirect Loop Troubleshooting Guide

A practical guide to diagnosing and fixing WordPress redirect loops, `ERR_TOO_MANY_REDIRECTS`, HTTPS conflicts, incorrect site URLs, proxy configuration problems, and server-level redirect rules.

## Table of Contents

- [Overview](#overview)
- [Common Symptoms](#common-symptoms)
- [1. Diagnose the Redirect Chain](#1-diagnose-the-redirect-chain)
- [2. Check WordPress Site URLs](#2-check-wordpress-site-urls)
- [3. Troubleshoot HTTP-to-HTTPS Redirects](#3-troubleshoot-http-to-https-redirects)
- [4. Diagnose Cloudflare and Reverse Proxy Issues](#4-diagnose-cloudflare-and-reverse-proxy-issues)
- [5. Review Apache and LiteSpeed Rules](#5-review-apache-and-litespeed-rules)
- [6. Review Nginx Configuration](#6-review-nginx-configuration)
- [7. Troubleshoot Login Redirect Loops](#7-troubleshoot-login-redirect-loops)
- [8. Check Plugins and Themes](#8-check-plugins-and-themes)
- [9. Diagnose Canonical Redirects](#9-diagnose-canonical-redirects)
- [10. Use Safe Diagnostic Commands](#10-use-safe-diagnostic-commands)
- [Recovery Checklist](#recovery-checklist)
- [Security and Production Best Practices](#security-and-production-best-practices)
- [Official References](#official-references)

## Overview

A redirect loop occurs when a browser or HTTP client is redirected repeatedly without reaching the intended destination.

WordPress websites can experience redirect loops because of conflicting site URLs, HTTPS settings, CDN rules, reverse proxy configuration, web server directives, plugins, or custom redirect code.

A common browser message is:

```text
ERR_TOO_MANY_REDIRECTS
```

The message alone does not identify the cause. Inspect the redirect chain and determine which configuration layer is generating the conflicting responses before changing settings.

## Common Symptoms

| Symptom | Possible cause |
|---|---|
| `ERR_TOO_MANY_REDIRECTS` | Conflicting redirect rules |
| HTTP redirects to HTTPS, then back to HTTP | SSL detection or proxy configuration issue |
| `www` redirects to non-`www` and back | Conflicting canonical hostname rules |
| WordPress login repeatedly reloads | URL, cookie, plugin, or HTTPS configuration |
| Homepage works but `/wp-admin/` loops | Login or administrative HTTPS rules |
| Redirect starts after enabling a CDN | CDN and origin redirect conflict |
| Redirect starts after migration | Incorrect WordPress URLs or stale rules |
| Only one URL or path loops | A path-specific plugin or server rule |

These are possible causes rather than definitive diagnoses.

## 1. Diagnose the Redirect Chain

Before editing any files, identify the URLs and status codes in the redirect sequence.

### Using browser Developer Tools

1. Open the affected URL.
2. Open Developer Tools.
3. Select the Network tab.
4. Enable **Preserve log**, if available.
5. Reload the page.
6. Inspect the document requests and their `Location` response headers.
7. Record the sequence of URLs and status codes.

Look for patterns such as:

```text
http://example.com/
    -> https://example.com/
    -> http://example.com/
    -> https://example.com/
```

This is an illustrative example, not a diagnosis of every redirect loop.

### What to investigate

- Which URL issues the first redirect?
- Does the protocol change repeatedly?
- Does the hostname alternate between `www` and non-`www`?
- Does the redirect happen before WordPress executes?
- Does it occur only after login?
- Does the redirect chain change when the CDN is bypassed through an authorized test?

The answers help narrow the investigation to WordPress, the web server, the proxy, or another layer.

## 2. Check WordPress Site URLs

WordPress has two important URL settings:

- **WordPress Address (URL):** Where the WordPress core files are located.
- **Site Address (URL):** The public address visitors use to reach the site.

For a typical installation where both use the same domain, the values might be:

```text
WordPress Address: https://example.com
Site Address:      https://example.com
```

The correct values depend on the installation, especially when WordPress core files are stored in a separate directory.

### Check the WordPress dashboard

If you can access the dashboard:

1. Open **Settings → General**.
2. Review WordPress Address and Site Address.
3. Confirm the expected protocol and hostname.
4. Save changes only if a value is incorrect.

### Check `wp-config.php`

Look for existing definitions of `WP_HOME` and `WP_SITEURL`:

```php
define( 'WP_HOME', 'https://example.com' );
define( 'WP_SITEURL', 'https://example.com' );
```

These are examples only. Do not add them blindly or duplicate existing definitions. These constants override the corresponding database options.

If the website uses a separate WordPress core directory, do not assume that both values should be identical.

### Check with WP-CLI

If WP-CLI is installed and you have authorized access to the correct installation, run:

```bash
wp option get home
wp option get siteurl
```

If needed, specify the installation path:

```bash
wp --path=/path/to/wordpress option get home
wp --path=/path/to/wordpress option get siteurl
```

Replace the example path with the actual WordPress directory.

Back up the database before changing URL options. Avoid performing broad search-and-replace operations merely to fix a redirect loop.

## 3. Troubleshoot HTTP-to-HTTPS Redirects

HTTPS redirects may be configured in WordPress, a plugin, the web server, a hosting control panel, or a CDN.

If multiple layers enforce conflicting rules, the browser can enter a loop.

### Troubleshooting steps

1. Confirm that the SSL certificate is valid for the domain.
2. Determine where HTTPS redirection is configured.
3. Check the WordPress Address and Site Address values.
4. Review `.htaccess` or server-level redirect directives.
5. Check any SSL or redirect plugins.
6. Review CDN or reverse proxy SSL settings.
7. Retest after correcting the confirmed conflict.

### Important distinction

A redirect from HTTP to HTTPS is usually expected. A loop occurs when another layer sends the HTTPS request back to HTTP or otherwise redirects it to a conflicting URL.

Do not disable HTTPS as a permanent workaround. Correct the configuration so the intended HTTPS URL is recognized consistently.

Official reference: https://developer.wordpress.org/advanced-administration/security/https/

## 4. Diagnose Cloudflare and Reverse Proxy Issues

When a CDN or reverse proxy terminates HTTPS before forwarding requests to the origin, WordPress may not automatically recognize the original visitor's connection as HTTPS.

This can cause WordPress and the proxy to disagree about the expected URL scheme.

### Checks for Cloudflare or similar services

1. Confirm the public SSL/TLS configuration.
2. Review whether the origin server has a valid certificate where required by the selected configuration.
3. Check for conflicting redirect rules at the CDN and origin.
4. Inspect the redirect chain and response headers.
5. Review whether the proxy forwards the original protocol correctly.
6. Verify the origin's behavior using an authorized, controlled test.

Avoid changing multiple SSL settings simultaneously.

### Reverse proxy considerations

A reverse proxy may send an `X-Forwarded-Proto` header indicating the original protocol. WordPress can use proxy-related information only when the setup is configured appropriately.

If the proxy terminates HTTPS and communicates with the origin over HTTP, configure HTTPS detection according to the official WordPress guidance and your proxy's trusted-header behavior.

Do not blindly trust forwarded headers from arbitrary clients. Ensure that the origin is protected so untrusted requests cannot spoof proxy metadata.

Reference: https://developer.wordpress.org/advanced-administration/security/https/

## 5. Review Apache and LiteSpeed Rules

Apache and LiteSpeed configurations may contain redirect rules in `.htaccess`, virtual host settings, or hosting control panel options.

### Troubleshooting steps

1. Back up the existing `.htaccess` file.
2. Review custom HTTP-to-HTTPS rules.
3. Check for repeated or contradictory hostname redirects.
4. Look for rules added by plugins or migration tools.
5. Compare custom directives with the standard WordPress rewrite block.
6. Test on staging where possible.
7. Restore the original file if a change causes additional problems.

### Example of a redirect conflict

A server-level rule may redirect all traffic to HTTPS while a plugin or proxy-related rule redirects requests back to HTTP.

The solution is not necessarily to remove every redirect. Identify the conflicting rule and keep one consistent redirect strategy.

Do not replace the entire `.htaccess` file with a generic template without preserving legitimate custom directives.

For LiteSpeed hosting, check the web server configuration and any cache or security plugin settings that may affect redirects.

## 6. Review Nginx Configuration

Nginx deployments may enforce redirects in server blocks, reverse proxy locations, or included configuration files.

### Troubleshooting steps

1. Identify the server block handling HTTP.
2. Identify the server block handling HTTPS.
3. Check `server_name` values.
4. Review `return` and `rewrite` directives.
5. Check proxy headers if requests are forwarded to PHP or another upstream.
6. Inspect Nginx and PHP-FPM logs.
7. Validate the configuration before applying changes.

If you have authorized server access, a configuration test can be run with:

```bash
sudo nginx -t
```

Only reload Nginx after the configuration test succeeds and the change has been reviewed.

Avoid copying generic server blocks into production without adapting them to the existing virtual host and TLS setup.

## 7. Troubleshoot Login Redirect Loops

If the public website loads but the WordPress login page or dashboard keeps redirecting, investigate authentication-related settings separately.

### Steps to try

1. Clear cookies for the affected domain.
2. Test in a private browser window.
3. Check `WP_HOME` and `WP_SITEURL`.
4. Verify the `home` and `siteurl` database options.
5. Review HTTPS and proxy detection.
6. Check security, login, caching, and redirect plugins.
7. Review PHP logs for authentication or session errors.

### Test plugin conflicts safely

If the dashboard is unavailable, an authorized administrator may temporarily deactivate a suspected plugin through a staging environment, WP-CLI, or the hosting file manager.

For example, with WP-CLI:

```bash
wp plugin list
```

Inspect the plugin list first. Avoid disabling all plugins on a live production website without considering the impact on security, checkout, membership, and other critical features.

Reference: https://developer.wordpress.org/advanced-administration/security/logging-in/

## 8. Check Plugins and Themes

Redirect behavior can be modified by:

- SSL plugins
- Redirect management plugins
- Security plugins
- Caching plugins
- SEO plugins
- Custom themes
- Custom code in `functions.php`
- Must-use plugins

### Safe conflict-testing procedure

1. Back up the website.
2. Reproduce the problem on staging.
3. Record the current redirect chain.
4. Test suspected plugins one at a time.
5. Test with a standard WordPress theme if appropriate.
6. Repeat the original request after each change.
7. Confirm the responsible component before changing production.
8. Restore the intended configuration.

If the loop disappears after a change, repeat the test to verify the cause rather than assuming it was fixed.

## 9. Diagnose Canonical Redirects

WordPress may redirect requests to a preferred URL for consistency. SEO plugins and custom code can also add canonical redirects.

A canonical redirect is not inherently an error. The problem occurs when different rules disagree about the correct destination.

### Check for conflicts involving

- Trailing slashes
- `www` versus non-`www`
- Uppercase and lowercase paths
- HTTP versus HTTPS
- Old and new domain names
- Custom redirect plugins
- Redirects configured during migration

Use the browser Network panel to identify the first redirect that points to the wrong destination.

Do not disable all WordPress canonical behavior simply because one URL is looping. Correct the specific conflicting rule instead.

## 10. Use Safe Diagnostic Commands

### Inspect the redirect chain

```bash
curl -sS -D - -o /dev/null \
  --max-redirs 10 \
  -L \
  https://example.com/
```

This follows redirects and prints response headers. Replace the example URL with the affected address.

### Display only response headers

```bash
curl -sS -D - -o /dev/null https://example.com/
```

### Inspect a specific URL without following redirects

```bash
curl -sS -I https://example.com/
```

Some servers handle `HEAD` requests differently from normal `GET` requests. If the result is inconclusive, inspect a normal GET request with `curl -D - -o /dev/null`.

### Inspect redirect status codes and destinations

```bash
curl -sS -L \
  --max-redirs 10 \
  -o /dev/null \
  -w 'Final URL: %{url_effective}\nHTTP status: %{http_code}\nRedirects: %{num_redirects}\n' \
  https://example.com/
```

These commands help inspect the public response chain. They do not automatically identify which configuration file or plugin caused the problem.

## Recovery Checklist

- [ ] Record the exact URL that loops.
- [ ] Capture the redirect chain and `Location` headers.
- [ ] Check WordPress Address and Site Address.
- [ ] Review `WP_HOME` and `WP_SITEURL`.
- [ ] Verify the SSL certificate and HTTPS configuration.
- [ ] Check CDN and reverse proxy settings.
- [ ] Review `.htaccess`, LiteSpeed, Apache, or Nginx rules.
- [ ] Investigate login cookies and authentication settings.
- [ ] Test plugin and theme conflicts on staging.
- [ ] Back up files and the database before changes.
- [ ] Retest HTTP, HTTPS, `www`, and non-`www` variants as applicable.
- [ ] Confirm that the intended URL loads without a loop.
- [ ] Document the root cause and the verified fix.

## Security and Production Best Practices

- Keep HTTPS enabled for sites that use it.
- Never publish credentials or private server configuration.
- Back up `.htaccess`, server configuration, and the database before changes.
- Avoid broad redirect exclusions that weaken security.
- Do not trust unverified forwarded headers.
- Avoid changing several redirect layers at once.
- Test configuration changes on staging when possible.
- Use a maintenance window for potentially disruptive production changes.
- Remove temporary debugging rules after troubleshooting.
- Verify that login, checkout, APIs, and other critical routes still work after the fix.

## Official References

- [WordPress: Logging In and Redirect Troubleshooting](https://developer.wordpress.org/advanced-administration/security/logging-in/)
- [WordPress: HTTPS Configuration](https://developer.wordpress.org/advanced-administration/security/https/)
- [WordPress: Migrating a Site](https://developer.wordpress.org/advanced-administration/upgrade/migrating/)
- [WordPress: Loopback Requests](https://developer.wordpress.org/advanced-administration/wordpress/loopback/)
- [WordPress: Site URL Function Reference](https://developer.wordpress.org/reference/functions/site_url/)

## Contributing

Contributions are welcome.

When submitting an improvement:

1. Describe the redirect symptom and affected environment.
2. Include sanitized response headers or redirect chains.
3. Explain how the root cause was identified.
4. Document the fix and any limitations.
5. Include relevant WordPress, PHP, proxy, and web server details.
6. Never include passwords, API keys, cookies, or private customer information.

## License

This repository can be distributed under the MIT License. Add a `LICENSE` file containing the MIT License text if you choose this license.

---

**Disclaimer:** This is an independent troubleshooting resource and is not an official WordPress project. Verify the root cause before making production changes, and maintain a recoverable backup.
