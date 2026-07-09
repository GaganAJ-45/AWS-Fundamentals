# CloudFront + S3 (Secure Static Hosting)

This documents upgrading a public S3-hosted static website to a production-style setup: **S3 (private) → CloudFront (CDN + HTTPS) → Visitors**, using **Origin Access Control (OAC)** so the bucket itself stays fully locked down.

Related: see [S3.md](./S3.md) for the base S3 concepts (buckets, storage classes, permissions, versioning) and the original static website hosting setup.
---

## Why Move From Plain S3 Hosting to CloudFront

The S3 website endpoint (`http://bucket-name.s3-website-region.amazonaws.com`) I used earlier works, but has real limitations:
- **HTTP only** — no HTTPS support
- **Single region** — visitors far from the bucket's region get slower load times
- **No caching layer** — every request hits S3 directly
- **Bucket must be public** — anyone can hit the bucket URL directly, bypassing any "official" domain

CloudFront fixes all four: it's a global CDN, gives free HTTPS, caches content at edge locations near visitors, and — with OAC — lets the bucket go back to fully private while still serving the site.

---

## Step 1: Make the Bucket Private Again

Since CloudFront (via OAC) will be the only thing allowed to read the bucket, undo the public setup from the earlier hosting steps:

1. Bucket → **Permissions** tab
2. **Bucket policy** → Edit → delete the existing public-read policy → Save (leave empty for now)
3. **Block public access (bucket settings)** → Edit → re-check **"Block all public access"** → Save → confirm

The site will "go down" temporarily until CloudFront is fully set up — expected.

---

## Step 2: Create a CloudFront Distribution

CloudFront is a separate service from S3 (it's global, not region-scoped). Find it via the top search bar → type **"CloudFront"** → select it. Then:

1. Click **Create distribution**
2. **Origin type**: select **Amazon S3**
3. **S3 origin**: pick the bucket — it should auto-fill as the REST API endpoint, e.g.
   ```
   gaganaj-portfolio.s3.us-east-1.amazonaws.com
   ```

   ⚠️ **Important**: AWS shows a yellow warning box suggesting *"Use website endpoint"* since the bucket has static hosting enabled. **Do not click that.** OAC only works with the S3 REST API endpoint. Switching to the website endpoint would force the bucket to stay public, defeating the purpose of this setup.

4. **Origin path**: leave empty (files sit at bucket root)
5. **Origin access**: select **Origin access control settings (recommended)** → **Create new OAC** → keep defaults → Create
6. **Enable Origin Shield**: No (extra caching layer, not needed for a personal site)
7. Click **Next** through **Enable security** (default settings are fine for a personal site)
8. On **Review and create**, confirm:
   - **Grant CloudFront access to origin: Yes** — confirms OAC is wired up correctly
   - A blue info box should say CloudFront will write/update the S3 bucket policy to restrict access to itself
9. Click **Create distribution**

Status will show **"Deploying"** — takes 5–10 minutes to roll out to edge locations globally.

---

## Step 3: Set the Default Root Object

This tells CloudFront what file to serve when someone visits the root `/` path (equivalent to the "Index document" setting in S3 static hosting).

If it wasn't set during creation:
1. Open the distribution → **General** tab → **Edit**
2. Find **Default root object**
3. Enter `index.html`
4. Save changes

---

## Step 4: Confirm the Bucket Policy Was Updated

CloudFront should auto-write a new bucket policy restricting access to only itself.

1. Go to the bucket → **Permissions** → **Bucket policy**
2. Confirm a policy now exists, referencing the CloudFront distribution's origin access identity/service principal
3. If it's still empty, go back to the CloudFront distribution's **Origin** section and use the **Copy policy** button, then paste it into the bucket policy manually

---

## Step 5: Test It

Once status changes from "Deploying" to **Enabled**, copy the **Distribution domain name** (e.g. `https://d20cosxjc33soy.cloudfront.net`) and open it.

Checks:
- Page loads over **HTTPS** automatically — no certificate setup needed for the default `*.cloudfront.net` domain
- Full site renders correctly (styles, images, all sections)
- Visiting the **old S3 website endpoint URL directly** should now fail — this confirms the bucket itself is properly locked down and only reachable through CloudFront

---

## Result — What This Actually Achieved

**Before:** S3 bucket (public) → visitors, HTTP only, single region, no caching
**After:** S3 bucket (private) → CloudFront (OAC + HTTPS + global caching) → visitors

This is the real-world production pattern companies use for static sites — the bucket is never exposed directly, and it's what I'd describe in an interview as: *"Deployed a static site using S3 + CloudFront with Origin Access Control, migrating from public bucket hosting to a secured CDN-backed architecture."*

---

## Optional Next Step: Custom Domain + HTTPS Certificate

Only relevant if a domain name is owned (e.g. from GoDaddy, Namecheap, or Route 53 registration). Not done yet — noting the steps for later:

1. **ACM (AWS Certificate Manager)** — must be requested in **us-east-1** specifically (CloudFront requirement, regardless of where the bucket lives) → request a public certificate for the domain → validate via DNS
2. **CloudFront** → edit distribution → add the domain under **Alternate domain names (CNAMEs)** → attach the ACM certificate
3. **Route 53** (or DNS provider) → create a record pointing the domain to the CloudFront distribution

---

## Key Lessons From This Setup

1. CloudFront + OAC needs the **S3 REST API endpoint**, not the website endpoint — ignore AWS's suggestion to switch if OAC is the goal
2. Granting CloudFront access during creation lets it **auto-write the correct bucket policy** — no need to hand-write an OAC policy from scratch
3. **Default root object** in CloudFront is the equivalent of S3's "Index document" — easy to miss during creation, but fixable anytime from the General tab
4. The end state should make the **direct S3 URL stop working** — that's the actual proof the bucket is private and CloudFront is the only path in

