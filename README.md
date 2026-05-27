# travismurdock.com

Personal site — single static page (Astro). Deployed via AWS Amplify Hosting from `main`.

## Stack
- **Framework:** Astro 6 (static output, no JS shipped to client)
- **Hosting:** AWS Amplify Hosting → CloudFront
- **DNS:** Route 53 (zone `Z10106222RMKMU12D5NGK`)
- **Email:** Google Workspace (MX/SPF/DKIM records preserved in same zone)

## Local development

```sh
npm install
npm run dev      # http://localhost:4321
npm run build    # outputs ./dist/
npm run preview  # serve built site locally
```

Requires Node ≥ 22.12 (see `.nvmrc`).

## Deploy
Pushes to `main` trigger Amplify build automatically. Build config: `amplify.yml`.
