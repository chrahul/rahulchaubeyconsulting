# rahulchaubeyconsulting.com

Homepage v1 — static site built with Astro.

## Run locally

    npm install
    npm run dev        # http://localhost:4321
    npm run build      # outputs static files to dist/

## Structure

    src/
      layouts/BaseLayout.astro   page shell, SEO and Open Graph tags
      components/Header.astro    navigation, including mobile menu
      components/Topology.astro  hero drawing
      components/Footer.astro
      pages/index.astro          homepage
      styles/global.css          design tokens and base styles
    public/favicon.svg

## Deployment

Static output (`dist/`), hosted on Hostinger.

Every push to `main` runs `.github/workflows/deploy.yml`: GitHub builds the site
and uploads `dist/` to Hostinger over FTP. No manual upload is needed.
It needs four repository secrets: FTP_SERVER, FTP_USERNAME, FTP_PASSWORD and
FTP_DIR (see the comments at the top of the workflow file). To redeploy without
a code change, run the workflow from the Actions tab ("Run workflow").

## Open items before publishing

- Course link points at the LMS root. Replace COURSE_URL in
  src/pages/index.astro with the exact course page.
- Contact is LinkedIn only until the email address is decided.
- Open Graph image not yet created.
