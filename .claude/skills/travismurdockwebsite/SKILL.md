---
name: travismurdockwebsite
description: Operating manual for travismurdock.com — Travis Murdock's personal website. Use when Travis asks to change, deploy, redeploy, debug, monitor, or take down travismurdock.com, or when he asks about its hosting, DNS, certificate, IAM role, GitHub repo, or build pipeline. Self-contained; does not depend on the EGG team.
owner: Travis Murdock (travis@travismurdock.com)
created: 2026-05-27
---

# travismurdock.com — operations skill

Single-page personal site. Signature logo + mailto link. Astro 6 static export → AWS Amplify Hosting → CloudFront. Pushes to `main` auto-deploy in ~45 seconds.

**Live:** https://travismurdock.com and https://www.travismurdock.com (HTTPS, Amazon-managed cert auto-renews)

---

## 1. Where everything lives

| Asset | Location |
|---|---|
| Source repo (GitHub, public) | https://github.com/travisrmurdock/travismurdock.com |
| Local working copy (Travis's Mac) | `~/code/travismurdock.com` |
| Amplify app (us-west-2) | `d2m15nxpbp2cbt` — https://us-west-2.console.aws.amazon.com/amplify/apps/d2m15nxpbp2cbt |
| Route 53 hosted zone | `Z10106222RMKMU12D5NGK` (in Travis's AWS account `727361761616`) |
| TLS cert | Amazon-managed via ACM, attached to Amplify domain, wildcard `*.travismurdock.com` |
| IAM role for DNS writes | `AWSAmplifyDomainRole-Z10106222RMKMU12D5NGK` — only Route 53 perms on that one zone, trusted by `amplify.amazonaws.com` |
| AWS account | `travis@travismurdock.com` root login — account ID **727361761616** |
| Domain registrar | Register.com / Network Solutions — expires **2027-12-02** (no action needed; NS already point at Route 53 in this account) |

---

## 2. How to change the site

Single page lives at `src/pages/index.astro`. Logo is `src/assets/logo.png` (Astro auto-generates 4 responsive WebP variants at build).

```sh
cd ~/code/travismurdock.com
npm install              # first time only
npm run dev              # http://localhost:4321 — live reload
npm run build            # produces ./dist/ (used by Amplify)
npm run preview          # serve the production build locally
```

**Requires Node ≥ 22.12** (specified in `.nvmrc`).

To deploy a change:

```sh
git add -A
git commit -m "..."
git push origin main     # triggers Amplify auto-build; ~45s to live
```

Watch the build at https://us-west-2.console.aws.amazon.com/amplify/apps/d2m15nxpbp2cbt/branches/main/deployments

---

## 3. DNS — what you can touch, what you can't

Route 53 zone `Z10106222RMKMU12D5NGK` contains records for both the website AND Travis's Google Workspace email.

**Website records (safe to modify if needed):**
- `travismurdock.com` A ALIAS → `d1uc2uejw9gsqa.cloudfront.net` (Amplify's CloudFront)
- `www.travismurdock.com` A ALIAS → same CloudFront target
- `_155c0f42e9f74314d0a4680fb0888bef.travismurdock.com` CNAME → ACM cert validation (don't delete, even though it's "done" — ACM uses it for renewal)

**🚨 EMAIL RECORDS — DO NOT TOUCH 🚨**
These power Travis's Google Workspace email at `travis@travismurdock.com`. Changing or deleting any of them breaks email.

- MX records (5×): `aspmx.l.google.com`, `alt1`, `alt2`, `alt3`, `alt4` (priorities 1, 5, 5, 10, 10)
- TXT `travismurdock.com`: `"v=spf1 include:_spf.google.com ~all"` (SPF)
- TXT `google._domainkey.travismurdock.com`: Google DKIM public key

**Other pre-existing records (legacy, leave alone unless cleaning up):**
- `blog.travismurdock.com` CNAME → `ghs.google.com` (old Google Sites blog)
- `gl2pe3taau54.travismurdock.com` CNAME → `gv-wpcbousdslk6ot.dv.googlehosted.com` (old Google site verification)

---

## 4. The Amplify gotcha (if you ever have to re-wire the domain)

When wiring a custom domain on a brand-new AWS account, the Amplify console UI shows status `SSL configuration: Adding subdomains records to your DNS provider...` and can sit there forever. The misleading bit: **Amplify is not actually writing the records** — it's stuck in `AWAITING_APP_CNAME` because the IAM role it needs doesn't exist and auto-creation silently failed.

**To diagnose:**
```sh
aws amplify get-domain-association --app-id d2m15nxpbp2cbt --domain-name travismurdock.com --region us-west-2
```
If `domainStatus` is `AWAITING_APP_CNAME` and `subDomains[].dnsRecord` lists a CloudFront target, Amplify is waiting on YOU to write the records.

**To fix (one-shot):**
1. Create the IAM role if missing:
   ```sh
   aws iam create-role --role-name AWSAmplifyDomainRole-Z10106222RMKMU12D5NGK \
     --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"amplify.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
   aws iam put-role-policy --role-name AWSAmplifyDomainRole-Z10106222RMKMU12D5NGK --policy-name AmplifyRoute53Access \
     --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["route53:ChangeResourceRecordSets","route53:ListResourceRecordSets","route53:GetChange"],"Resource":"arn:aws:route53:::hostedzone/Z10106222RMKMU12D5NGK"},{"Effect":"Allow","Action":["route53:ListHostedZones","route53:ListHostedZonesByName","route53:GetHostedZone","route53:GetChange"],"Resource":"*"}]}'
   ```
2. Get the CloudFront target from `get-domain-association` (the `dnsRecord` field) — looks like `d<random>.cloudfront.net`.
3. UPSERT the A records to ALIAS pointing at it (CloudFront's hosted zone ID is always `Z2FDTNDATAQYW2`):
   ```sh
   aws route53 change-resource-record-sets --hosted-zone-id Z10106222RMKMU12D5NGK --change-batch '{
     "Changes":[
       {"Action":"UPSERT","ResourceRecordSet":{"Name":"travismurdock.com","Type":"A","AliasTarget":{"HostedZoneId":"Z2FDTNDATAQYW2","DNSName":"<cloudfront-target>","EvaluateTargetHealth":false}}},
       {"Action":"UPSERT","ResourceRecordSet":{"Name":"www.travismurdock.com","Type":"A","AliasTarget":{"HostedZoneId":"Z2FDTNDATAQYW2","DNSName":"<cloudfront-target>","EvaluateTargetHealth":false}}}
     ]
   }'
   ```
4. Site is live within ~60 seconds (Route 53 propagation).

**Caching gotcha:** local macOS resolver may keep returning the old IP for several minutes even when querying authoritative NS directly. Always cross-check with `dig +short A travismurdock.com @1.1.1.1` and `@8.8.8.8` — those reflect truth faster than the local resolver during a cutover.

---

## 5. How to monitor / verify

```sh
# Site live?
curl -sI https://travismurdock.com/ | head -5

# Returns real Astro page (not Amplify placeholder)?
curl -s https://travismurdock.com/ | grep '<title>'   # should be "<title>Travis Murdock</title>"

# DNS pointing at CloudFront?
dig +short A travismurdock.com @1.1.1.1               # expect 108.138.x.x range (CloudFront)

# Email still flowing (MX intact)?
dig +short MX travismurdock.com @1.1.1.1              # expect 5 aspmx.l.google.com etc.

# Cert details?
echo | openssl s_client -servername travismurdock.com -connect travismurdock.com:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

---

## 6. Costs

- Route 53 hosted zone: $0.50/mo (was already in account)
- Amplify Hosting (static, near-zero traffic): ~$0.01–0.15/mo
- ACM certificate: free
- **Total new spend: under $1/mo**

---

## 7. Build configuration

`amplify.yml` at repo root:
```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - nvm use 22
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

Auto-detected by Amplify; produces static files in `dist/` which Amplify serves via CloudFront.

---

## 8. Quick reference

| Want to… | Do this |
|---|---|
| Edit the page content/styles | Edit `~/code/travismurdock.com/src/pages/index.astro`, push to `main` |
| Replace the logo | Replace `~/code/travismurdock.com/src/assets/logo.png` (Astro re-optimizes at build), push to `main` |
| Check deploy status | Open https://us-west-2.console.aws.amazon.com/amplify/apps/d2m15nxpbp2cbt |
| Roll back | In Amplify console → Deployments → click an older successful deployment → "Redeploy this version" |
| Take site down | In Amplify → Hosting → Pause builds, or delete domain association (DNS records stay) |
| Add a subdomain | Amplify → Custom domains → Add new subdomain (will auto-write Route 53 record IF the IAM role exists — see §4) |

---

## 9. What this skill does NOT cover

- Email setup (Google Workspace lives outside this; see Travis's Google Admin console)
- Other AWS resources in the account (e.g., the old EC2 at `35.164.204.112` that previously served the placeholder nginx — orphaned, can be terminated if found)
- Any EGG team workflows — this is independent
