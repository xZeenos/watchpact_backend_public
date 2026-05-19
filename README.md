# WatchPact — Media Tracking App (Backend)

![Tests](https://github.com/xZeenos/watchpact_backend/actions/workflows/test.yml/badge.svg)

> This repository is a portfolio showcase. The source code is kept private.

This was my first larger scale personal project. I worked on it on and off for 4 years with a good friend who is building the Android app. The Watchpact App (currently being ported from native Java to Flutter for a cross-platform release) is very much a passion project for both of us — to learn project management and how to write production-ready code, and is also something that family and friends have been asking us to make.

**Website:** https://watchpact.com  
**Google Play Store:** https://play.google.com/store/apps/details?id=com.watchpact.navdar

**Tech stack:** Python · Flask · PostgreSQL · Redis · Docker · Nginx · SvelteKit

## Purpose and features

WatchPact is a backend (with a small SvelteKit frontend for a web presentation) for a media tracking app. It uses the TMDB API as a data provider.

Features:
- User accounts with JWT authentication
- Group system with invite codes — share a watchlist with friends
- Add movies and series to a group, mark them as watched, add a personal description and genre tag
- Search for media or browse popular/upcoming titles via TMDB
- Collection feature — add an entire TMDB collection (e.g. all films in a franchise) in one go
- Streaming provider support — see where you can watch each title, and filter by the providers you're subscribed to
- Cache check endpoint — the app only re-fetches data when something has actually changed

## Architecture

I always found it kind of intimidating when peers and friends talked about all these different frameworks and found it confusing which and how to choose. I think this architecture is pretty decent for what it needs to be doing. This project helped me grow a lot when it came to what good architecture might look like.

The architecture very much expanded once I got more and more into it. Now in its probably final state it's a Docker Compose based setup. There are 6 different containers running on prod and 5 when dev is running. Secret management is handled using Infisical and there is an UptimeRobot check running on the health endpoint.

The basis is a Postgres DB with a Flask API (using Flask-RESTful and Flask-ApiSpec for Swagger docs) sitting in front of it, running behind Gunicorn with 18 workers in prod. Flask and Nginx communicate over a Unix socket instead of a TCP port, which felt like a cool little detail to add once I learned that was a thing since it reduces network overhead. Redis handles the JWT blocklist for logouts and any caching. On top of that there is a cron-job container that runs scheduled DB maintenance, and a Restic container that does automated backups to a Backblaze B2 bucket so I don't lose everyone's data. Nginx handles SSL termination with Let's Encrypt auto-renewal baked in.

Looking back I am happy with how it turned out, even if I'd do some things differently today (see Limits below).

## Security

Security was something I tried to put real emphasis on — partly because real users' data is running on this, and partly because I wanted to learn what production-grade security actually looks like in practice.

- **JWT token blocklist:** Tokens are invalidated server-side in Redis on logout. The client discarding a token is not enough — a stolen token would otherwise remain valid until expiry.
- **Rate limiting:** Sensitive endpoints are rate-limited via Flask-Limiter — registration at 20 req/min, login at 10 req/min — to slow down brute-force and credential stuffing attempts.
- **App API key:** Registration and login require a shared app secret in the request header, so the API isn't openly accessible to anyone who finds the URL.
- **Password reset tokens:** Signed with itsdangerous and time-limited, so reset links expire and can't be forged.
- **Nginx security headers:** HSTS with preload, CSP, X-Frame-Options, X-Content-Type-Options, Permissions-Policy, COEP, and COOP headers are all set on the production Nginx config.
- **Secret management:** All secrets are managed via Infisical and injected at runtime — no plaintext secrets in the repo or in committed env files.
- **Non-root containers:** All Docker containers run as unprivileged users.

## CI/CD and reliability

Production deployment is handled automatically via GitHub Actions: every push to `main` that passes the test suite triggers a deployment to the production server over SSH. There is also a manual trigger available from the Actions tab (requires typing `deploy` to confirm). Dependabot is configured for pip, npm, Docker, and GitHub Actions dependencies on a weekly schedule.

The test suite uses pytest with mocked dependencies so it runs without Docker. The deployment script rebuilds containers, verifies all services are healthy, and tests the database connection before finishing.

## Limits

This is very much only one of my passion projects and I want to try out so many more things. Since I tend to not finish projects if I don't accept certain facts and decisions I made — because then it is not perfect — there are some known limits I want to mention.
For a future project I really want to switch to an even better setup using Django with an ORM data model. The cache should use an MD5 hash, and for password encryption bcrypt is kind of outdated — argon2 is way better.

## AI Usage

Towards the end of this project I started using Claude Code to help accelerate certain tasks — mainly the refactoring of a monolithic `app.py` (~3000 lines) into Flask Blueprints, adding tests and their respective mocks. The architecture, design decisions, data model, and the bulk of the code are the product of four years of iteration. My goal using it is very much based on challenging myself to learn how to handle the new reality developers are faced with. Knowing what to delegate, reviewing what it produces, and integrating it into a real workflow, while ensuring my own understanding and control during the development.

## Disclaimer

We are in no way affiliated with TMDB and our use of their API is in no way endorsed or an endorsement of this project. 
