# Note when using Cloudflare & Letsencrypt

Letsencrypt needs to renew its certificate every 60 days. However when the site is behind Cloudflare, its being proxied and "protected" by Cloudflare. So you need to make a Cloudflare rule in order to let the renewal happen.

In Cloudflare go to "Page Rules" for the domain. Make sure there is a rule set with the following content:

`URL: *devops.foundation/.well-known/acme-challenge/*`

Settings then are:
`SSL` is `Off`
