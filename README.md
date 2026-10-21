# Zivo cab booking
Run: `mvn spring-boot:run`, then open http://localhost:8080. The admin login is printed on first start.
Env vars: `DB_URL` `DB_USER` `DB_PASS` (PostgreSQL), `JWT_SECRET` (32+ chars, keeps logins valid across restarts), `ADMIN_NAME`, `ADMIN_EMAIL`, `ADMIN_PASSWORD` (choose your admin login; if an admin exists and `ADMIN_PASSWORD` is set, that admin is updated at start).

## Before launch
1. Fill the `{{...}}` placeholders in `privacy.html` and `terms.html`, and have both reviewed by a lawyer.
2. Custom domain: point your domain's A record at the server, install `deploy/nginx.conf` (replace `{{DOMAIN}}`), run certbot for HTTPS.
3. Run `./check-launch.sh` until it passes.
4. Stripe PaymentIntents are supported by the backend. Set `STRIPE_SECRET_KEY` using a test key first. `POST /api/rides/{id}/pay` creates an intent and `POST /api/rides/{id}/pay/confirm` verifies it server-side. The simple frontend currently creates the intent but does not yet include a full Stripe Elements card form, so payments are not production-ready until Stripe.js/Elements is connected to the returned `clientSecret`.

Paid rides expose a PDF receipt at `GET /api/rides/{id}/invoice`. Ratings accept optional written comments; read driver reviews at `GET /api/drivers/{id}/reviews`. Admin promo endpoints are `POST/GET /api/admin/promos`; riders can validate with `POST /api/promos/validate`. Promo validation is not yet automatically applied to fare calculation. SMTP environment settings are included but automatic email delivery is not enabled.

## Docker
```
cp .env.example .env     # then edit the three values
docker compose up --build
```
Open http://localhost. This starts PostgreSQL, the app and Nginx. Admin login: `admin@cab.local` with your `ADMIN_PASSWORD` (if left empty, see `docker compose logs app`). Data is kept in the `pgdata` volume. For HTTPS on a real domain use `deploy/nginx.conf`. GitHub Actions (`.github/workflows/ci.yml`) builds the jar and the image on every push.

## Map tiles
Tiles come from OpenStreetMap's public servers, which is fine for development and light use. For real traffic, switch to a keyed provider (MapTiler, Stadia) or host your own tiles.


## Payment gateway setup (test mode)

Zivo uses Stripe for online payments. Configure Stripe test credentials as environment variables before starting Spring Boot; never commit real secrets.

- `STRIPE_SECRET_KEY` = Stripe test secret key (`sk_test_...`)
- `STRIPE_PUBLISHABLE_KEY` = Stripe test publishable key (`pk_test_...`)

On Windows PowerShell, set these for the current terminal with `$env:STRIPE_SECRET_KEY="sk_test_..."` (repeat for the other variables), then run `mvn spring-boot:run` from the folder containing `pom.xml`. In IntelliJ, configure the same values in Run > Edit Configurations > Environment variables. Restart the app after changing them.

When a completed ride is awaiting payment, click **Pay** and complete the Stripe test checkout using Stripe test card details. Stripe confirmation retrieves the PaymentIntent server-side and checks its succeeded status, amount, ride ID, and rider ID. For production, also configure signed provider webhooks, HTTPS, persistent payment records/idempotency, refunds, and reconciliation before accepting live payments.
