# Loop & Roam Records — The Case of the Accidental `.env`

**Category:** BASICS  
**Difficulty:** EASY  
**Solves:** 65+  
**Flag:** `WEBVERSE{3ed9db31...d227c4}`

---

## The Setup

Loop & Roam Records is an indie label born in a Detroit garage in 2011. Their site is maintained by Aphra, a contract designer who was learning git as she built it.

One night in February 2024, she committed a `.env` file packed with production credentials. She caught the mistake an hour later, ran `git rm .env`, and pushed. Problem is, `git rm` only removes a file from the *current* commit — the file lives forever in the commit history. And to make matters worse, the deploy script `rsync`s the entire working tree to the web root — **including the `.git/` directory**.

---

## Recon

The lab site is a clean, minimal Bandcamp-style record label page. Nothing unusual on any of the frontend pages (`/`, `/roster`, `/releases`, `/tour`, `/store`). No hidden comments, no suspicious cookies, no secrets in response headers.

But the challenge briefing drops a heavy hint: Aphra committed a `.env`, removed it, and the `.git/` directory is on the public web root.

A quick check confirms:

```
GET /.git/ → 200 OK ✅
```

The `.git/` directory is fully exposed and browsable.

---

## Exploitation

With the `.git/` directory accessible, we can download everything and inspect the commit history. The key is that `git rm .env` only removes the file from the *working tree and future commits* — the `.env` contents still exist in any commit where it was previously staged.

### Step 1: Map the Commit History

The git reflog (`/.git/logs/HEAD`) reveals the full timeline:

```
5448e99 initial: flask skeleton + readme + minimal home
f4c75a9 style: design system
a71ace4 templates: full base layout
a8624ef home: hero with single-of-the-week
bd42d23 wip: wire up prod env file           ← 🔴 .env ADDED here
c420919 oops: remove .env from tracking       ← 🟢 .env "removed" here
b8b5627 content: roster + releases pages
499e148 content: tour + store pages
0d50bfd readme: full pages list, deploy notes
```

Commit `bd42d23` ("wip: wire up prod env file") is where the `.env` was introduced.

### Step 2: Extract the `.env`

Downloading and decompressing that commit's tree object reveals a `.env` file packed with credentials:

```
NODE_ENV=production
DATABASE_URL=postgres://looproam:Pq7d2-vT83-bWuM@db.internal...
SHOPIFY_API_KEY=shp_a8e72b3f4d
STRIPE_SECRET_KEY=sk_live_4920ab48f5d2e9a4c0
MAILGUN_API_KEY=key-7f4ea9c82b3d0a1e6c5d8e7f
BANDCAMP_OAUTH_TOKEN=bc-oauth-49281acd-...
INTERNAL_REF=WEBVERSE{3ed9db31...d227c4}     ← 🚩
```

The `INTERNAL_REF` field holds the flag.

---

## Key Takeaways

| Lesson | Why It Matters |
|--------|----------------|
| **`git rm` is not a delete** | It only removes a file from the current HEAD — the commit history retains everything. Use `git filter-branch` or `BFG Repo-Cleaner` to truly purge secrets. |
| **`.git/` does not belong in production** | `rsync` without a `.gitignore`-style exclude will copy the entire `.git/` directory. Use `.rsync-filter` or explicitly exclude it in your deploy script. |
| **`.env` files are not just "don't commit"** | Even if you catch it quickly, the secret has already left your local machine and entered the repo's history. |

**Remediation checklist:**
- [ ] Expose `.git/` → `curl -s https://target.com/.git/config` — if 200, you have a problem
- [ ] Rotate any secrets that were committed, even briefly
- [ ] Set up pre-commit hooks (`detect-secrets`, `talisman`, `git-secrets`) to catch secrets before they land in a commit

---

*"Aphra learned git the hard way. Now you get the XP."*

**Flag:** `WEBVERSE{3ed9db31...d227c4}`
