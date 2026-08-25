# coach-report

Static page for workout session reviews. One file, no build step.

**This repo is a deploy target, not the source.** The canonical file lives in the
private `health-native` repo at `web/coach-report/index.html`, versioned next to
the edge function it talks to. Edit it there and run `web/deploy-report-page.sh`;
edits made here will be overwritten.

## Why it is public

The page holds no secrets. It contains only the Supabase project URL and the
publishable key — both public by design — and fetches everything through
`/functions/v1/coach-report/{token}`. The credential is the token in the report
link, so the file itself is safe to serve to anyone.

Nothing here can reach a report without a token, and each token is scoped to a
single session.
