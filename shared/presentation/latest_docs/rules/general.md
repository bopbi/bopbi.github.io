# General

## ⚠ Never Commit Automatically

**Never run `git commit` (or any git write command) without explicit user instruction.**

This includes staging files (`git add`), committing, pushing, or amending. Always stop after making code changes and wait for the user to say "commit".

## ⚠ Security Constraint — No User-Supplied Image URLs

**Do not re-enable image display from user-supplied URLs under any circumstances.**

This includes:
- Re-adding `image_urls` to `generic_info_post` or `community_info`
- Removing the `html.SkipImages` flag from the markdown renderer (`lib/markdown/markdown.go`)
- Adding any new form field or DB column that accepts external image URLs from users

**Why:** External image URLs submitted by users are a security risk — they can deliver tracking pixels that leak visitor IP addresses, carry malicious payloads exploited by browser image parsers, and load content from untrusted third-party servers. The `SkipImages` renderer flag and the removal of `image_urls` from the schema are intentional, permanent decisions.

**If image support is needed in the future:** images must be uploaded to a controlled, self-hosted storage (e.g. object storage behind the same domain) and served from there — never loaded from a URL typed in by the user.
