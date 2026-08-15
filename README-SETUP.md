# Launch checklist — from zero to live blog

Total time: about 45 minutes. Total cost: ~$10–15/year (the domain).
Do the steps in order.

## 1. Fix up your GitHub profile: github.com/zugate-real (10 min)

1. Settings → Public profile → set **Name** to "Kumar Shubham" (right now
   the profile shows no name, so recruiters searching you find nothing),
   add a photo (same as LinkedIn), location "Bengaluru, India", and a bio:
   "Senior iOS engineer · Swift/SwiftUI · platform & frameworks".
2. Create a public repo named exactly `zugate-real` (so the full path is
   zugate-real/zugate-real) and add the provided github-profile-README.md
   to it as README.md — it renders on your profile page automatically.

## 2. Create the blog repository (5 min)

1. New repository → name it exactly: `zugate-real.github.io`.
   Public. No template.
2. Upload everything in this folder (except this README) to the repo root:
   `_config.yml`, `index.html`, `about.md`, `_layouts/`, `_drafts/`,
   `assets/`. Easiest way without git installed: repo page → "uploading an
   existing file" link → drag the folder contents in → Commit.
3. Repo → Settings → Pages → confirm Source is "Deploy from a branch",
   branch `main`, folder `/ (root)`.
4. Wait ~2 minutes. Your site is now live at
   `https://zugate-real.github.io` — check it renders.

## 3. One config check (1 min)

Everything is pre-filled with your username (zugate-real). Only edit
`_config.yml` if you buy a different domain than kumarshubham.dev —
change the `url:` line to match.

## 4. Buy the domain (10 min)

1. Go to **porkbun.com** (or Namecheap — either is fine, both ~$10–15/yr
   for `.dev`). Search `kumarshubham.dev`. Fallbacks if taken:
   `kumarshubham.blog`, `kshubham.dev`, `shubham.engineering`.
   Note: `.dev` domains are HTTPS-only, which is exactly what you want.
2. Buy it. Skip every upsell (hosting, email, SSL — you need none of it).

## 5. Point the domain at GitHub Pages (10 min)

In your registrar's DNS settings for the domain, add these records:

| Type  | Host | Answer |
|-------|------|--------|
| A     | (blank / @) | 185.199.108.153 |
| A     | (blank / @) | 185.199.109.153 |
| A     | (blank / @) | 185.199.110.153 |
| A     | (blank / @) | 185.199.111.153 |
| CNAME | www  | zugate-real.github.io |

Delete any default "parked" A/CNAME records the registrar pre-filled.

Then: repo → Settings → Pages → Custom domain → enter `kumarshubham.dev`
→ Save. Wait for the DNS check (can take from minutes up to an hour),
then tick **Enforce HTTPS**.

## 6. Publish the first post (when you're ready — not today)

The JSONDecoder article is in `_drafts/`. Before publishing:

1. **Run both repro snippets yourself** on your Mac. Verify the depths
   at which they crash on your setup and adjust the numbers in the post
   if yours differ. Never publish a repro you haven't run.
2. Read the whole draft and make it sound like you. Edit freely — it's
   a draft, not a script.
3. To publish: move the file from `_drafts/` to `_posts/` and rename it
   with a date prefix:
   `_posts/2026-08-23-your-jsondecoder-is-one-payload-away-from-a-stack-overflow.md`
4. Commit. It's live ~2 minutes later at
   `kumarshubham.dev/your-jsondecoder-is-one-payload-away-from-a-stack-overflow/`

## 7. Distribution (the day after publishing)

In this order:
1. **Swift Forums** (forums.swift.org) — post in "Related Projects" or
   "Using Swift" with a 2–3 paragraph summary + link. Engage with replies.
2. **iOS Dev Weekly** — iosdevweekly.com, use the link submission form.
3. **Hacker News** — submit the URL with the post's exact title, no spin.
4. **r/iOSProgramming** and **r/swift** — link post, then answer comments.
5. **LinkedIn** — short human post (3–4 sentences, one takeaway), link in
   the first comment or inline.
6. **Mastodon** — post with #iOSDev.

Wait a day between publishing and submitting so any typo reports from
early readers get fixed before the traffic spike.

## Ongoing

- New post = new file in `_posts/` named `YYYY-MM-DD-slug.md` with the
  same front matter as the first one. That's the entire workflow.
- RSS feed exists automatically at `/feed.xml`; sitemap at `/sitemap.xml`.
- After the first post is live, add the blog URL to your GitHub profile,
  LinkedIn, and the resume header.
