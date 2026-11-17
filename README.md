# DigitalDuniya360

A responsive static website for an India-focused community, marketplace and business directory. The frontend is HTML, CSS and vanilla JavaScript modules; Supabase provides authentication, PostgreSQL, Row Level Security and Storage. There is no build step.

## Run locally

1. Open this folder in VS Code.
2. Start a local static server from the project root (ES modules do not work reliably from `file://`):

   ```powershell
   py -m http.server 8000
   ```

   If Python is not installed, use the VS Code Live Server extension.
3. Open `http://localhost:8000`.

The public pages and browser-based tools work without a Supabase project. Database-backed content intentionally shows empty states until configured; there are no invented listings, testimonials, deals, job postings or engagement counts.

## Supabase setup

1. Create a Supabase project.
2. The exact supplied project URL and publishable key are configured in `assets/js/config.js`. Confirm those values under **Project Settings → API** if you use a different Supabase project. Both values are public browser configuration; never add a secret/service-role key to any frontend file.
3. Open **SQL Editor**, paste and run the complete `database/schema.sql`. It creates the relational tables, indexes, RLS policies and image buckets. The site's connection indicator will report when the database schema is available.
4. Run `database/community-offers.sql` to enable signed-in users to publish product deals and coupon offers immediately, link Amazon, Flipkart, Meesho or another HTTPS website, and upload one product image. For existing projects, run this migration after the main schema.
5. For existing projects, run `database/affiliate-click-delete-fix.sql` after the main schema. It makes deleting a deal or coupon also delete its related affiliate click records, avoiding a conflict with the click table's exactly-one-offer constraint.
6. In **Authentication → URL Configuration**, set **Site URL** to `https://digitalduniya360.com` and add these Redirect URLs (replace with your actual domain if different):
   - `http://localhost:8000/**`
   - `https://digitalduniya360.com/**`
   - `https://www.digitalduniya360.com/**` (if the www domain is enabled)
   Configure email verification and SMTP before public launch. Password reset returns to `/login.html`.
7. Register your own account and verify its email. Then run the commented admin bootstrap SQL at the bottom of `database/schema.sql`, replacing `you@example.com` with that account's email. This trusted-owner step is manual; registration never grants admin access. Use the admin panel to grant or revoke trusted admin/moderator roles. Only admins may manage roles.
8. Publish content through the admin moderation panel. Public listings query only published records; news also requires the verified flag.

## Paid membership system setup

The membership system adds a ₹9 introductory plan (one-time, 60 days, available once during the first 72 hours after registration) and a ₹499 monthly Razorpay subscription (up to 12 billing cycles). The monthly plan charges automatically each cycle after the user authorizes the subscription; users can cancel future cycles from **My Account**, while access already paid for remains until the current period ends. Each signed-in account gets one free submission in each supported category. Existing attributed submissions are backfilled as already-used credits; anonymous classified ads cannot be attributed to an account. Free credits do not reset when a membership expires.

### Apply the database migration

Run these scripts in the Supabase SQL Editor, in this order:

1. `database/schema.sql`
2. `database/community-offers.sql`
3. For an existing installation, `database/affiliate-click-delete-fix.sql` and `database/classified-ads-rls-fix.sql`
4. `database/membership-system.sql` **last**

The membership migration adds tables and policies without deleting existing content. It backfills existing submissions as used credits and restricts future submissions to the atomic `submit_content_with_access` RPC. Always rerun the membership migration after a later change to a posting policy; rerunning the community-offers or classified-ads policy migrations after it could reopen a direct insert route. Review the migration against a database backup/staging project before production.

### Configure and deploy Supabase Edge Functions

Install the Supabase CLI, run `supabase login`, and link this folder to the correct Supabase project:

```powershell
supabase link --project-ref YOUR_PROJECT_REF
supabase functions deploy membership-payment
supabase functions deploy razorpay-webhook
supabase functions deploy admin-console
```

In the Supabase Dashboard's Edge Function secrets, configure `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET`, `RAZORPAY_STANDARD_PLAN_ID`, and `ALLOWED_ORIGINS` (`https://digitalduniya360.com,https://www.digitalduniya360.com`). `RAZORPAY_STANDARD_PLAN_ID` is the Razorpay Plan ID created for the ₹499 monthly plan. Confirm the hosted function environment also provides `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY`; add them only to the server-side secret store if they are not supplied automatically. Never place any of these secrets in browser JavaScript, the Hostinger public directory, source control, or chat. `supabase/config.toml` disables JWT verification only for the Razorpay webhook; payment and admin functions still require authenticated Supabase users.

Use Razorpay **Test Mode** first. Create one Razorpay Plan in Test Mode for ₹499 (`amount: 49900` paise, `period: "monthly"`, `interval: 1`); the Edge Function creates subscriptions against this Plan with `total_count: 12`. Copy its `plan_...` ID into the server-side `RAZORPAY_STANDARD_PLAN_ID` secret. Create the plan through Razorpay's dashboard/API using a trusted server environment—never add the Razorpay secret to browser code. Do not create a separate plan for each user.

In the Razorpay Test Mode dashboard, create a webhook pointing to `https://YOUR_PROJECT_REF.supabase.co/functions/v1/razorpay-webhook`, use a long random webhook secret, and subscribe to `payment.captured`, `payment.failed`, `subscription.authenticated`, `subscription.activated`, `subscription.charged`, `subscription.pending`, `subscription.halted`, `subscription.cancelled`, `subscription.completed`, `subscription.paused`, and `subscription.resumed`. Put that webhook secret in Supabase Edge Function secrets. The handler verifies raw-body webhook signatures and records event IDs to make duplicate deliveries idempotent. One-time introductory payments are verified against their order; subscription checkout signatures and captured charges are verified against Razorpay before membership periods are recorded. Subscription lifecycle webhooks keep cancellation and renewal status synchronized.

For Live Mode, complete Razorpay account activation and domain verification, create the ₹499 monthly Plan again in Live Mode, and use its Live Mode Plan ID. Switch to Live Mode API keys and create a separate Live Mode webhook with a separate webhook secret. Replace the corresponding Supabase Edge Function secrets, verify the live webhook URL and events, and run a low-value authorized production transaction before announcing the service. Never enable live keys in the frontend.

### Create the first administrator

There is no public admin registration. In Supabase **Authentication → Users**, create or invite your administrator email and set its password through Supabase. After confirming the exact account email, run this statement in SQL Editor, replacing the example address:

```sql
insert into public.admin_roles (user_id, granted_by)
select id, id
from auth.users
where lower(email) = lower('you@example.com')
on conflict (user_id) do nothing;
```

If no row is inserted, verify that the Auth user already exists and the email matches. Keep the password in Supabase Auth; do not hardcode it in the site or SQL file. Existing admin roles from `public.user_roles` are synchronized by the migration. Admin access is checked by Supabase on the server for both `/admin-login` and every dashboard function request.

### Routes and Hostinger

`/pricing`, `/my-account`, and `/admin-login` are rewritten to static HTML entry points by the root `.htaccess`; `/admin/` uses `admin/index.html`. Upload the `.htaccess` file together with the site and ensure Hostinger's Apache rewrite module is enabled. Keep HTTPS enabled, then directly open and refresh each route after deployment. The pricing page is public; account and admin entry pages carry `noindex`, and the dashboard itself returns no administrative data unless the signed-in user has the admin role.

The ₹9 introductory payment is a one-time Razorpay order. The ₹499 plan creates a Razorpay subscription on the server and, after user authorization, charges monthly for at most 12 cycles unless cancelled. Cancellation stops future charges and does not remove access already paid for. A frontend success callback alone never activates a membership: the backend verifies the subscription signature and captured payment, and signed webhooks are the durable source for later cycles. Prices are stored as integer paise and membership periods use Razorpay billing timestamps recorded by the database.

### Membership test checklist

Use a staging Supabase project and Razorpay Test Mode before production:

- Register fresh test users and check the first 72-hour offer, the server-side expiry boundary, one-time introductory purchase and ₹499 monthly subscription availability after expiry.
- Verify one free submission for each category, then a second same-category submission is rejected until membership is active. Test two simultaneous submissions, logout/login and another browser/device.
- Confirm pre-existing attributed content is counted, anonymous older ads stay unattributed, and free credits do not reset after expiry.
- Exercise successful ₹9 payment and verified 60-day activation; subscribe to the ₹499 monthly plan, verify the first captured cycle and subsequent `subscription.charged` cycle, cancel future renewals and confirm access remains through the already-paid period.
- Test failed subscription charges, subscription pending/halted/completed events, duplicate webhook deliveries, an uncompleted checkout and a user who already has an active subscription.
- Confirm ordinary users cannot read other accounts' payment/membership data, call admin actions, alter credits or activate membership; confirm an authorized admin can sign in, inspect records, change content/account status and export CSV.
- Re-test existing login, posting, classified listings, blogs, jobs, businesses and admin moderation.

No Supabase or Razorpay credentials are stored in this project, so database execution, webhook delivery, subscription creation/cancellation, payments and live admin authorization must be verified in your own staging project before launch.

### Hostinger deployment

1. In Hostinger hPanel, open **Websites → Manage → File Manager** (or connect with FTP).
2. Upload the project contents—not an enclosing project folder—into the domain's `public_html` directory. Keep the existing relative paths such as `assets/js/` and `database/`.
3. The supplied Supabase Project URL and publishable key are in `assets/js/config.js`. The publishable key may be visible to site visitors; that is expected. Do not upload `.env` files, a database password, or a service-role key.
4. Enable SSL in hPanel and confirm `https://digitalduniya360.com` loads. If the domain points at a subfolder or uses another canonical host, adjust the existing canonical URLs and `sitemap.xml` accordingly.
5. Test login, email verification, password recovery, uploads, posting, ad auto-publication and anonymous ad/photo submission on the live HTTPS domain. Ensure the exact live origin and redirect URLs are allow-listed in Supabase Authentication.

This static site does not have a private runtime environment-variable store: `assets/js/config.js` is its public configuration module. If you want to avoid committing public configuration to Git, upload that file directly through Hostinger File Manager after deployment. The service-role key must remain server-side and is not required by the website.

### Storage

The schema creates `blog-images`, `deal-images`, `classified-images`, `business-images`, `user-avatars`, and `news-images`. Buckets are public-read for images, limited to JPG/PNG/WebP and 5 MB per file. Other upload paths begin with the signed-in user's UUID; classified ad photos can be uploaded anonymously under an ad-specific path. Never store private documents in these public buckets.

### Integrated application surfaces

- Public home, blog/news/classified/deal/coupon/business/job/community listings and detail routes.
- SEO resources, blog publishing, product listing, blog commenting and article submission guides are available at `/seo-services/`, `/blog-platform/`, `/product-listing/`, `/blog-commenting/` and `/article-submission/`.
- The homepage includes an editorial overview and accessible FAQ; SEO landing pages include unique metadata, canonical URLs, social previews and structured breadcrumbs/FAQs.
- Branded favicon assets and a custom `404.html` page are included. The root `.htaccess` redirects the `www` host, rewrites the membership routes, and maps missing paths to the custom error page; verify these behaviors after deployment. Hostinger currently upgrades HTTP to HTTPS at its edge, so configure or verify the canonical host redirect in hPanel to avoid a two-step redirect for `http://www`.
- Signup, login, email verification, login/logout, password reset and reset-password completion, private dashboard, profile editing, avatar upload and password change.
- Pending submission forms for blogs (required website URL and featured image), businesses (multiple images), jobs and community questions; classified ads publish immediately without login; signed-in users can publish product deals and coupon offers with one image immediately; blog comments, business reviews and community answers are moderated.
- Owner-scoped submission status and delete actions for supported account submissions; classified ads are managed by administrators.
- Database-verified admin queues for blogs, news, businesses, jobs, questions, coupons, comments, reviews, answers and reports, plus published-ad and community-offer unpublish/delete controls, deal/coupon entry and role management.
- Public listings for published content; community offers open their merchant link directly and disclose possible affiliate commissions; news additionally requires administrator verification.
- Fourteen in-browser utilities on `seo-tools.html`; image work stays in the browser. The basic site checker cannot bypass cross-origin browser restrictions and is not a full crawler/audit.
- Responsive navigation, page-specific titles/descriptions, Open Graph/Twitter metadata, organization/website/FAQ/breadcrumb structured data, noindex on private or detail routes, robots.txt and a static sitemap.

Before production, additionally implement server-verified quiz scoring and points, full poll voting UX, saved items and notifications, full content editing, email workflows, server-side rate limiting and abuse detection. Never trust a client-side action for payment, points or security decisions. The browser-only SEO checker does not perform an external crawl.

## Before launch checklist

- [ ] Confirm the supplied Project URL and publishable key in `assets/js/config.js` match the intended Supabase project.
- [ ] Run `database/schema.sql` in the Supabase SQL Editor (including on existing projects) and verify RLS with anonymous, ordinary-user and admin accounts.
- [ ] Run `database/community-offers.sql`; verify signed-in users can post and anonymous visitors cannot, including with an HTTPS Website link.
- [ ] For an existing installation, run `database/affiliate-click-delete-fix.sql` so deleting deals or coupons does not violate the affiliate-click constraint.
- [ ] For an existing installation, run `database/classified-ads-rls-fix.sql` to apply the focused ad submission policies without replacing other tables or records.
- [ ] Run `database/membership-system.sql` last after the base and existing-project policy migrations; verify its backfill counts before enabling paid posting.
- [ ] Configure the Razorpay Test Mode secrets in Supabase Edge Function secrets and deploy all three membership functions; verify the webhook delivery before considering Live Mode.
- [ ] Create the administrator through Supabase Auth and explicitly assign its UUID in `public.admin_roles`; test with a separate ordinary account.
- [ ] Directly refresh `/pricing`, `/my-account`, `/admin-login` and `/admin/` on Hostinger after uploading `.htaccess`.
- [ ] Configure email verification, password recovery redirects, SMTP and allowed origins for Hostinger's HTTPS domain.
- [ ] Upload the complete static project to Hostinger `public_html` and enable SSL.
- [ ] Create a trusted admin role using the documented manual bootstrap; verify an ordinary signup cannot access `/admin/`.
- [ ] Review all RLS rules and Storage policies against the production threat model.
- [ ] Test forms, anonymous ad/photo submission, immediate ad publication, admin unpublishing/deletion and public visibility with separate test accounts.
- [ ] Configure a moderation process, backups, privacy notice, terms, cookie/analytics choices and data retention.
- [ ] Set up a server-side/Edge Function layer for rate limits, quiz scoring, email workflows and any remaining privileged operations before enabling those features.
- [ ] Replace the example domain in canonical URLs and `sitemap.xml`, then submit the sitemap to search engines.

## Security notes

All user-generated text is rendered as escaped text. Classified ads publish immediately, including anonymous submissions; signed-in users can publish community offers immediately, and administrators can archive or delete them. Other user-submitted content remains pending by default. RLS protects offer insertion and permits public reads only for published offers. Merchant destinations and coupon terms are user-provided and are not guaranteed; inspect reports and remove misleading offers. Never expose the Supabase service-role key. Admin assignment is intentionally manual and must only be performed by a trusted project owner. Review upload/storage exposure and legal/privacy requirements before production use.
