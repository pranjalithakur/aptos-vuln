# Target Architecture — Juice Shop

> Suggested Document Base categorization: **kind = discovery**, **scope = repository** (linked to the `juice-shop/juice-shop` repository) or **application scope = selected applications** if uploaded directly against the Agentic Pentesting application.

## Target identifiers

- Source repository: https://github.com/juice-shop/juice-shop
- Test/staging instance: http://juice-shop.juice-shop.svc.cluster.local/ (internal cluster address — not the public OWASP demo instance)

## What this application is

OWASP Juice Shop is an intentionally vulnerable e-commerce web application maintained as an OWASP flagship project. It is built to exercise the full OWASP Top Ten plus additional real-world flaw classes, and is commonly used as a target for security tool evaluation, training, and CTFs. Treat every feature area below as a *candidate* for a real, findable vulnerability — that is the application's design intent, not an assumption.

## Tech stack

| Layer | Technology |
| --- | --- |
| Frontend | Angular (TypeScript SPA), served from `/dist` under the same origin as the API |
| Backend | Node.js + Express (TypeScript), REST API under `/api`, `/rest`, and `/socket.io` |
| Relational data | SQLite via Sequelize ORM |
| Document-style data | MarsDB (in-memory, MongoDB-like) — used for product reviews |
| Auth | JSON Web Tokens (bearer, `jsonwebtoken` + `express-jwt`) — no server-side session store |
| File handling | `multer` (multipart upload), a local file-serving/quarantine path, and a log-file server |
| API docs | `swagger-ui-express` exposes a live API schema |
| Logging | `morgan` request logging; app also exposes a `/metrics` endpoint |

## Core data model

- **User** — email, password hash, role, security question/answer, 2FA secret (optional)
- **Product** — catalog items, including a search endpoint
- **Basket / BasketItem** — per-user shopping cart
- **Order / Delivery** — checkout, order history, order tracking
- **Address / Card / Wallet** — shipping addresses, stored payment card records, and an internal wallet/loyalty-points balance
- **Feedback / Complaint** — user-submitted feedback and complaints; complaints accept file attachments
- **Coupon** — discount codes
- **SecurityQuestion / SecurityAnswer** — used for password-reset flows
- **Challenge / ChallengeDependency / Hint** — the app's own internal scoring/hint system (not customer-facing business data, but present in the same database)

## Functional areas / endpoint surface

Grouped by intent, not by literal route path — use this to scope reconnaissance rather than as an exhaustive endpoint list:

- **Authentication & account recovery**: login, 2FA, change password, reset password, security-question flow, continue-code (email verification style) flow
- **Profile & identity**: user profile read/update, profile image upload (both direct file upload and upload-by-URL)
- **Core shopping flow**: product search, basket, basket items, checkout/payment, delivery selection, coupon redemption, order history, order tracking
- **Social / user-generated content**: product reviews (create, update, like), chat, feedback, complaints (with file attachment)
- **B2B integration**: a B2B order endpoint that accepts XML input — a distinct trust boundary from the customer-facing JSON API
- **Loyalty / monetization extensions**: wallet, deluxe membership, premium rewards, NFT minting
- **Administrative & operational**: app configuration, metrics, data export, data erasure, progress restore, a "recycle" endpoint, a log-file server, a general file server, and a quarantine server for uploaded content
- **Miscellaneous**: redirect handling, video handling, an "easter egg" endpoint, and endpoints that intentionally expose vulnerable code snippets/fixes for training purposes

## Roles and privilege boundaries

- **Anonymous visitor** — browse catalog, search, view public product reviews
- **Registered customer** — the roles above plus basket/checkout, profile, feedback, complaints, order history
- **Deluxe member** — a paid tier with additional wallet/reward features layered on top of the customer role
- **Administrator** — reachable via a distinct admin section; has access to configuration, user management, and moderation actions

The customer → deluxe → admin boundary, and the separate B2B (XML) trust boundary, are the two privilege/trust-boundary axes most worth mapping during reconnaissance.

## Notes for scan planning

- Authentication is stateless (JWT bearer), so authorization checks live per-endpoint rather than in a central session gate — inconsistent enforcement across similar-looking endpoints is a realistic failure mode here, not a hypothetical one.
- The app deliberately mixes a relational store (SQLite/Sequelize) with a document-style store (MarsDB) for different data types — injection technique and payload should be considered per data path rather than assumed uniform across the app.
- File upload/serving is a first-class feature area (profile images, complaint attachments, a dedicated file server, and a "quarantine" server), making it a natural place to look for path traversal, content-type confusion, and unrestricted-upload issues.
