# mta-sts.jeffops.com

A one-file website. It publishes the MTA-STS policy for `jeffops.com` so that
sending mail servers require TLS when they deliver to the domain, instead of
falling back to plaintext when someone tampers with the connection.

The policy has to be served over HTTPS from `mta-sts.<domain>`, on a certificate
valid for that exact host. That is the whole reason this repository is separate
from the site: GitHub Pages allows one custom domain per repository, and
`jeffops.com` already belongs to the site repo.

## What is in here

    CNAME                    mta-sts.jeffops.com — claims the domain for this repo
    .nojekyll                stops Pages from discarding .well-known (see below)
    .well-known/mta-sts.txt  the policy itself
    index.html               a page for anyone who visits the host directly

`.nojekyll` is not optional. GitHub Pages runs Jekyll by default, and Jekyll
skips every file and directory whose name begins with a dot — which is exactly
where the spec requires the policy to live. Without this file the policy URL
returns 404 and every check silently reports "no policy".

## The policy

    version: STSv1
    mode: testing
    mx: *.mail.protection.outlook.com
    max_age: 604800

`mode: testing` means a sending server that cannot establish valid TLS reports
the failure and **delivers the mail anyway**. Nothing can be lost while this is
set. `mode: enforce` means it refuses to deliver instead. Move to enforce only
after the TLS-RPT reports have been quiet for a few weeks.

The `mx` pattern matches `jeffops-com.mail.protection.outlook.com`, which is the
MX for the domain. If the MX ever changes, this line has to change with it.

`max_age` is seven days: how long a sender caches the policy. It is deliberately
short while in testing and can go up to 604800 or beyond once enforcing.

## Changing the policy

Edit the file, then **change the id in DNS**. Senders only re-fetch when the id
changes:

    TXT  _mta-sts.jeffops.com   v=STSv1; id=<new value>

The id is an opaque string of at most 32 alphanumeric characters. A UTC
timestamp such as `20260803120000Z` is the usual convention and makes it obvious
when the policy last moved.

## Publishing this repository

1. Create a new public repository on GitHub — the name does not matter,
   `mta-sts` is fine. It must be public: GitHub Pages on a free account does not
   serve private repositories.
2. Push these files to its default branch.
3. Settings → Pages → Source: *Deploy from a branch*, branch `main`, folder `/`.
4. The custom domain fills itself in from the CNAME file. Tick
   **Enforce HTTPS** once the certificate is issued — this can take a few
   minutes, and it will fail until DNS has propagated.

The DNS half is already in place:

    CNAME  mta-sts    jeffwouters.github.io
    TXT    _mta-sts   v=STSv1; id=20260803120000Z
    TXT    _smtp._tls v=TLSRPTv1; rua=mailto:jeff@jeffops.com

## Checking it worked

    curl https://mta-sts.jeffops.com/.well-known/mta-sts.txt

That must return the policy over a valid certificate. Until it does, the DNS
records are inert — which is harmless, because a policy that cannot be fetched
is treated as no policy at all.
