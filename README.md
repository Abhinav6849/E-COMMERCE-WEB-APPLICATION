# MegaBazaar – Java Spring Boot backend + website

Spring Boot 3 (Java 17) · Spring Security + JWT · JPA/Hibernate · H2 (local) / PostgreSQL or MySQL (online)

## What you get

| Feature | Where |
|---|---|
| Sign up (account creation) – password stored as a **BCrypt hash**, never as plain text | `register.html`, `POST /api/auth/register` |
| Login with email + password, JWT token, 5 wrong attempts = 15 min lock | `login.html`, `POST /api/auth/login` |
| **Forgot password with email OTP** (6 digits, 10 min expiry, 5 tries, one-time use, 60 s resend) | `forgot-password.html`, `/api/auth/forgot-password`, `/verify-otp`, `/reset-password` |
| Customer dashboard: profile, change password, order history | `dashboard.html` |
| Admin dashboard: add/edit/remove **company products**, see all orders (change status), users, sales overview | `dashboard.html` (admin login) |
| Product catalogue stored in the database (16 starter products from your HTML) | `products.html`, `product.html`, `/api/products` |
| Cart → checkout → order saved in DB (prices & stock always checked on the server) | `checkout.html`, `POST /api/orders` |
| Page navigation: Home → Products → Product → Checkout → Dashboard | all pages share `js/app.js` |

## 1. Run on localhost

1. Install **JDK 17 or newer** and **Maven 3.9+** (or just open the folder in IntelliJ IDEA / VS Code / Eclipse – they bundle Maven).
2. In this folder run:

```
mvn spring-boot:run
```

3. Open **http://localhost:8080**

First start creates the database (`./data/megabazar.mv.db`) and these accounts/data automatically:

* Admin login: `admin@example.com` / `Admin@12345` (**local testing only – change before going online**)
* 16 products loaded from `seed-products.json`

### Try the flow
1. Click **Sign Up**, create a customer account → you land on your dashboard.
2. Add products to the cart → **Proceed to Checkout** → place an order (Cash on Delivery).
3. Dashboard → **My Orders**.
4. Logout, log in as admin → **Products** to add a product, **All Orders** to change order status, **Users** to see registered customers.
5. Forgot password: on the login page click *Forgot password?*. With no mail server configured, **the OTP is printed in the console** (look for `[DEV MODE] Password reset OTP`).

> Passwords cannot be viewed by anyone, not even the admin – only the secure hash is stored. That is intentional and is how real sites work.

### Send OTP by real email (Gmail example)
Turn on 2-Step Verification for the Gmail account, create an **App password**, then set these before starting:

```
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=youraddress@gmail.com
MAIL_PASSWORD=<the 16-character app password>
MAIL_FROM=youraddress@gmail.com
```
(Windows PowerShell: `$env:MAIL_HOST="smtp.gmail.com"` … then `mvn spring-boot:run`.)
For a live shop use a transactional email service (Brevo, Amazon SES, Resend …) with the same variables.
Also set `ADMIN_EMAIL` to your own real email so the admin can use forgot-password.

## 2. Build a single runnable file
```
mvn clean package
java -jar target/megabazar.jar
```

## 3. Put it on the internet

The app needs **three environment variables** online (the `prod` profile refuses to start without them):

| Variable | Meaning |
|---|---|
| `JWT_SECRET` | random text, 32+ characters (e.g. generate with a password manager) |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | your real admin login (created on first start) |
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | a PostgreSQL or MySQL database, e.g. `jdbc:postgresql://HOST:5432/DBNAME` (add `?sslmode=require` if the provider needs SSL) |
| `SPRING_PROFILES_ACTIVE=prod` | already set in the Dockerfile |
| `MAIL_*` | email settings from above (needed for OTP emails) |

Note: JDBC URLs start with `jdbc:`. Providers often show `postgres://user:pass@host/db` – convert it to `jdbc:postgresql://host:5432/db` and put user/password in `DB_USERNAME` / `DB_PASSWORD`.

### Option A – Render (easiest, uses the included Dockerfile)
1. Push this folder to a GitHub repository (`data/` and `target/` are already git-ignored).
2. On render.com: **New → Web Service** → pick the repo → Runtime **Docker**.
3. Add the environment variables above. Create a PostgreSQL database (Render's, Neon, or Supabase) and fill `DB_*`.
4. Deploy → you get `https://yourapp.onrender.com`.

Free-tier caveats (check Render's current terms): free web services sleep after ~15 minutes without traffic and need about a minute to wake up, and free Postgres databases expire after a limited period (reported as ~30 days) and are then deleted. For a real shop use a paid always-on instance and a database that does not expire, or a VPS.

### Option B – a VPS (always on, you control everything)
Any Ubuntu VPS (Hostinger, DigitalOcean, AWS Lightsail, Oracle Cloud …):
1. `sudo apt install openjdk-17-jre nginx certbot python3-certbot-nginx`
2. Upload `megabazar.jar`; run it as a systemd service with the environment variables above (you can use the H2 default by omitting `DB_*` and not using the `prod` profile, but PostgreSQL/MySQL is recommended).
3. Nginx reverse proxy: `proxy_pass http://127.0.0.1:8080;` (forward `Host`, `X-Forwarded-For`, `X-Forwarded-Proto`).
4. HTTPS: `sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com`

## 4. Domain name
1. Buy a domain from a registrar (GoDaddy, Hostinger, Namecheap, Porkbun, HIOX, Cloudflare …). `.in` and `.com` are common. **Compare the renewal price, not only the first-year offer** – many registrars discount year one heavily.
2. Add DNS records at the registrar:
   * **Render:** in the service → *Settings → Custom Domains*, add your domain; Render shows the exact `CNAME`/`A` records to create. HTTPS certificate is automatic.
   * **VPS:** an `A` record for `@` and `www` pointing to the server's IP address.
3. DNS can take from a few minutes to a few hours to spread.

## 5. Before real customers
* Change admin credentials, use a long random `JWT_SECRET`, serve only over HTTPS.
* Cash on Delivery is the only payment method now. For online payments integrate a gateway such as Razorpay (needs a business account) – orders already store totals and status.
* Back up the database regularly. Add Terms, Privacy Policy and refund pages (legal requirement for stores in India).

## API quick reference
```
POST /api/auth/register | /login | /forgot-password | /verify-otp | /reset-password
GET  /api/products?q=&category=&sort=price_asc|price_desc|rating&page=0&size=20
GET  /api/products/{id}      GET /api/products/categories
GET  /api/me     PUT /api/me     POST /api/me/password            (Bearer token)
POST /api/orders      GET /api/orders                              (Bearer token)
ADMIN: GET /api/admin/stats|users|products|orders, POST|PUT|DELETE /api/admin/products[/{id}],
       PATCH /api/admin/orders/{id}/status
```
Send `Authorization: Bearer <token>` for protected endpoints.

## Project layout
```
src/main/java/com/megabazar/{config,controller,dto,exception,model,repo,security,service}
src/main/resources/application*.properties, seed-products.json
src/main/resources/static/  index.html login.html register.html forgot-password.html
                            products.html product.html checkout.html dashboard.html js/app.js
```
