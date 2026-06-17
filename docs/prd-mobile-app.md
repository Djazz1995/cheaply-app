# PRD — Cheaply Mobile App (React Native)

_Local-business social-commerce app. Discover, follow, and walk into local/small businesses through their live products, deals, and reels._

Status: Draft
Owner: D. van der Hoogt
Date: 2026-06-17
Absorbs: the former `docs/prd-local-discovery-network.md` (positioning + Nextdoor analysis folded into §1a below).

---

## 1. Vision

A mobile-first social network **for small businesses to reach consumers** — local shops _and_ online-only webshops. For local businesses the goal is **get more people through the door**; for online shops it is **drive visits + outbound clicks to their store**. Same engine (follow + home/pages + push + offers), two outcomes.

It is also a **discovery platform for the businesses themselves** — not only their deals. The shop/brand is a first-class discoverable unit: consumers find, follow, and return to small businesses worth knowing **even when there's no active deal**. Deals-first stays the hook; business discovery is the relationship that outlasts any single offer.

Not a clone of Nextdoor's neighbor-chatter. The unit of content is the **business and its offers** (products, deals, reels, events), not neighbor gossip. Cheaply already owns the hard part Nextdoor lacks — real, structured local commerce inventory (verified businesses, geocoded locations, products, prices, discounts, validity, search). The app turns that inventory into a daily-habit discovery surface in people's pockets.

> **One line:** Instagram-meets-local-deals — every small business near you, on a map and in a feed, with their live products, deals, and short videos.

---

## 1a. Positioning & competitive context

> **Nextdoor** = discover local businesses through your neighbors (trust-first, ads-funded).
>
> **Cheaply** = discover local businesses through their live deals (commerce-first, deal-funded).

Same end value to a business — **be discovered by nearby people** — but Cheaply's unit of discovery is a verified current deal/offer, not a sponsored post or neighbor mention. Sharper hook for the business ("list your current offer free, reach nearby shoppers") and for the consumer ("see what's actually on sale near me right now").

**The strategic bet:** do **not** clone Nextdoor's verified-neighbor social graph (the hardest cold-start in consumer apps, highest moderation cost). Instead, layer a thin discovery + follow loop onto Cheaply's existing commerce inventory. Follows + deals + geo replicate enough of Nextdoor's discovery value without the social graph.

**Mapping (Nextdoor → Cheaply):**

| Nextdoor capability | Cheaply equivalent | Reuses |
|---|---|---|
| Verified neighbor graph | _Skipped_ — replaced by location pref + follows | geo, location cookie |
| Neighborhood feed | Local deal/offer feed (location + follow ranked) | products, Typesense, geo |
| Business Page | Business profile (§6.4) | companies, locations |
| Follow a business | Follow business / category | new follow model |
| "Good plumber near me?" | Category + nearby discovery | categories, geo facets |
| Local Deals (ad product) | Deals = the core object, native | products |
| Notifications | New-deal alerts for followed entities | email + push |
| Sponsored posts / Ads | Boosted/featured deal (Phase 3) | featured products |

**Incumbents:** Hoplr (NL/BE), Nebenan (DE, Burda-funded) own social-first hyperlocal in-region. Cheaply's wedge = commerce-first + deals + reels, business→consumer — must stay differentiated, not drift into a worse Hoplr.

**Web vs app:** the web storefront (former local-discovery PRD) stays as the SEO/GEO acquisition funnel + recurring web discovery surface; the app is the retention/habit + push + reels layer. Same backend. Don't kill web — it's top-of-funnel.

---

## 2. Why an app (and the honest tradeoffs)

**Why mobile:**
- Local discovery is inherently on-the-go (location, map, "what's near me right now").
- Push notifications are the retention engine local web can't match ("deal near you ends today").
- Reels/video are a native-mobile behavior.
- Camera access = businesses record reels in-app.

**Honest costs (not "easier"):**
- App Store + Play Store review, two platforms, native build pipeline, push certificates, store compliance.
- Updates gated by store review (mitigate with OTA via Expo EAS Update).
- **Mitigation: reuse the entire existing backend.** Payload CMS, PostgreSQL, Typesense, scraper, email — all stay. The app is a new client over an API layer, not a new system. This is what makes the scope tractable.

**Stack decision: React Native via Expo.**
- Expo (managed) → fastest path, OTA updates, easy push, camera/maps modules, single codebase iOS+Android.
- Reuse existing TypeScript skills + types from the web repo.

---

## 3. Goals & non-goals

### Goals
- Give every local/small business a presence consumers can discover, follow, and act on.
- **Make small businesses discoverable as the unit** — surface new/nearby/notable shops & brands, not only their deals. Discovery works even with no active offer.
- Drive the core business outcome: **foot traffic + outbound clicks/calls/directions.**
- Make discovery a **daily habit** via home + map + push, not one-shot search.
- Reuse Cheaply's backend and content automation (scraper-filled supply = no empty-room cold start).

### Non-goals (v1)
- Neighbor-to-neighbor social network (consumer-to-consumer posts/chat). Stay business→consumer.
- In-app payments / taking a transaction cut. App sends people to the business; deal happens there.
- Self-serve ads marketplace. Monetization is a later phase.
- Heavy UGC moderation pipeline beyond business-submitted reels.

---

## 4. Users & supply strategy

### Consumer
Lives/works in region. Wants relevant deals — nearby shops/restaurants **and** online webshops worth following — without effort. Opens app to browse home, check map, get push alerts.

### Business (two kinds, one platform)

Every business has a **type**: `local`, `online`, or `hybrid`.

- **Local** — physical shop/restaurant. Goal = foot traffic. Has location(s), map pin, directions/call.
- **Online** — webshop, no walk-in. Goal = store visits + outbound clicks. No location, no map pin; profile CTA = **Visit shop**.
- **Hybrid** — physical shop that also ships. Both surfaces.

All want reach. Three ways onto the platform (tiered — minimize effort):

| Tier | How listed | Business effort |
|---|---|---|
| **Auto** | Scraper / Shopify API / product feed pulls their catalog | **Zero** — listed without lifting a finger |
| **Connected** | Business connects Shopify/Woo → live product sync | One-time connect |
| **Manual / self-serve** | Business app/portal: add products, deals, reels, events | Ongoing, lightweight |

**Critical advantage:** supply is **not** user-generated-from-zero like Nextdoor. Scraper + commerce connectors auto-fill inventory, so home/pages/map are populated on day one. Online webshops are a *natural fit for auto/connected* — they already have Shopify/feeds, so onboarding is near-zero. Businesses are approached *after* they're already getting clicks ("you're getting X views, claim & boost").

The only true supply gap = pure-offline shops with nothing scrapable → manual self-serve.

---

## 5. Content types (what a business can have)

1. **Product listings** — scraped, Shopify/feed-connected, or manually entered. Price, image, link.
2. **Deal listings** — discount/promo with validity dates (already native in Cheaply).
3. **Restaurant deals/offers** — menu specials, happy hour, daily deals.
4. **Event / market ads** — "we're at [market] this Saturday," pop-ups, fairs. Time + place + map pin.
5. **Reels / short video** — business-submitted product/store videos. (Later phase — heavier infra.)
6. **Business profile** — name, type (local/online/hybrid), categories, locations + map pins (local/hybrid), hours/contact (local/hybrid), shop URL (online/hybrid), follower count.

---

## 6. Core app surfaces

### 6.1 Home (categorized overview)
- **Not one mixed feed** — a categorized landing surface. One section/row per content type, each a horizontal preview with **"See all" → dedicated page** (§6.1a):
  - **Deals** — top nearby/followed deals.
  - **Products** — new/relevant products.
  - **Reels** — short-video previews (Phase 3).
  - **Events** — upcoming nearby (Phase 2).
  - **Businesses** — new/nearby/trending shops to discover (no active deal needed).
- Personalized + location-scoped. **Local items** rank by distance + discount + recency + validity-ending-soon + follow-boost (Typesense geo sort). **Online items** (no distance) rank by relevance + discount + recency + validity + follow-boost.
- **Scope filter:** local / online / all (default all). Anonymous = location + online; identified = follow-personalized.
- Section order/visibility can adapt to personalization + supply (hide empty sections).

### 6.1a Dedicated content pages
Each Home section opens a full page — single content type, own filters/sort, infinite scroll:
- **Deals page** — all deals; filter category/scope/has-validity, sort distance/discount/ending-soon.
- **Products page** — all products; filter category/scope, sort relevance/recency/price.
- **Reels page** — vertical short-video feed (Phase 3); tap → product/business.
- **Events page** — upcoming events (Phase 2); list + map, filter by date/category.
- **Businesses page** — discover shops; filter category/scope/type, sort new/nearby/trending. (The business-discovery home.)

All reachable from Home "See all" and via deep links. Filters/scope shared with Discover where it makes sense.

### 6.2 Map
- **Local + hybrid** businesses as pins (existing geocoded company locations). **Online-only excluded** (no geo).
- Filter by category, open-now, has-active-deal, restaurants, events.
- Tap pin → business profile + current offers + directions.

### 6.3 Discover / search (query-driven)
- **Search across everything** — businesses, deals, products, events. Typesense-backed (categories, city facets, validity, geo sort). This is the query-driven entry; Home + dedicated pages (§6.1/6.1a) are the browse-driven entry.
- Category browse hubs (food, drinks, fashion, home, furniture, gifts) that funnel into the relevant dedicated page (filtered).
- Results typed (business vs deal vs product); scope filter (local/online/all) applies.
- **Curated collections (Phase 2):** editorial lists ("makers in Leiden", "new this week", themed) reusing existing admin/editorial + content automation. Low cost, high discovery value; not MVP-critical.

### 6.4 Business profile
- Products, deals, reels, events, follow button, follower count (social proof).
- **Local/hybrid:** locations/map, hours, contact. Actions: **Directions, Call, Visit, Follow.**
- **Online:** shop URL, ships-to. Actions: **Visit shop, Follow** (no directions/call).

### 6.5 Reels (later phase)
- Vertical short-video feed of business-submitted clips. Tap → product/business.

### 6.6 Saved & Following
- Saved deals/products + followed businesses. A return reason.

### 6.7 Business mode (self-serve)
- Separate area (or companion flow) for businesses to manage profile, add products/deals/events, upload reels, connect Shopify, see basic stats (views, clicks, directions taps).

---

## 7. Notifications (retention engine)
- **Phase 1 (followed-only):** new deal from a followed business/category; deal ending soon. Driven by follow relationships only — predictable, high-relevance, protects opt-in.
- **Later (proximity-based):** new business/reel near you; event this weekend nearby. Deferred — noisier, harder opt-in, needs careful frequency tuning before shipping.
- Volume control: smart batching + user-set frequency. Default conservative to protect opt-in.
- Push infra via Expo push.

---

## 8. Architecture

```
┌─────────────────────────────┐
│   React Native app (Expo)   │  iOS + Android
│  home · pages · map ·       │
│  profile · reels · biz mode │
└──────────────┬──────────────┘
               │ REST/GraphQL (JSON)
┌──────────────┴──────────────┐
│   API layer (Next.js API /  │  thin, mostly exists
│   Payload REST + custom)    │
├─────────────────────────────┤
│ Payload CMS 3  ·  Postgres  │  REUSED — already built
│ Typesense (search + geo)    │  REUSED
│ Scraper + Shopify/feed sync │  REUSED (supply)
│ Nodemailer + Expo Push      │  email reused, push new
│ Media/video storage + CDN   │  NEW (reels)
└─────────────────────────────┘
```

**Reused (already in repo):** Payload collections (products, companies, locations, categories, blog, media, users), Typesense search/geo, scraper, postcode/geo, email, localization data.

**New build:**
1. **Consumer identity** — passwordless email / social login (no end-user accounts today, only admin).
2. **Follow + saved** relationships.
3. **Public app API** — clean read endpoints over Payload/Typesense for the app (auth, home + content pages, map, search, profile, follow).
4. **Push** — Expo push tokens + send pipeline hooked to product-save/Typesense-sync hooks.
5. **Business self-serve** — auth'd business accounts, product/deal/event/reel CRUD, Shopify connect, basic analytics.
6. **Reels** — video upload, storage, transcode, CDN, feed. (Later phase.)
7. **Events/market ads** — new content type (time + geo + business).

---

## 9. What's genuinely new vs reused

| Capability | Status |
|---|---|
| Product/deal data, search, geo, map data | ✅ Reuse |
| Scraper / Shopify / feed supply | ✅ Reuse (extend with platform-aware ingestion) |
| Email infra | ✅ Reuse |
| Consumer accounts, follow, saved | 🆕 Build |
| Public app API | 🆕 Build (thin) |
| Push notifications | 🆕 Build |
| Business self-serve portal | 🆕 Build |
| Events / market ads | 🆕 Build |
| Reels / video | 🆕 Build (later phase) |

---

## 10. Phasing

**Phase 1 — Consumer MVP (read-only + follow).**
- Expo app: home (categorized) + dedicated pages (deals/products/businesses), map, discover/search, business profile.
- **Business discovery:** business row on Home + dedicated Businesses page (new/nearby/trending shops), follow with no deal required. Woven into existing surfaces (no new tab).
- Consumer identity (passwordless), follow, saved.
- Push for followed-business deals.
- Supply = scraper/connectors (zero business effort). Includes **online webshops** (Shopify/feed = near-zero onboarding) alongside local.
- Goal: prove daily-habit retention in Leiden region (local) + follow-driven online discovery.

**Phase 2 — Business self-serve.**
- Business accounts, claim listing, manual products/deals/events, Shopify connect, basic stats.
- Events / market ads content type.
- **Curated collections** — editorial discovery lists (reuse admin/editorial + content automation).

**Phase 3 — Reels + monetization.**
- Business reels/short video.
- Paid: boosted deals / featured placement / priority feed. (The Nextdoor "ads" wedge, Cheaply-flavored.)

---

## 11. Success metrics
- DAU/WAU and **return rate** (daily-habit proof, not one-shot).
- Outbound business actions: directions taps + calls (local "in the door" proxy); shop-visit clicks (online conversion proxy).
- Follows per active user; followers per business.
- **Business discovery:** profile views per business; follows originating from discovery (Home business row / Businesses page / Discover, not deal-gated); % of follows to businesses with no active deal; new-business profile reach.
- Push opt-in % and notification CTR.
- Businesses auto-listed vs claimed vs self-serve-active.
- Live deals / businesses per region (supply density).

---

## 12. Risks & open questions

- **Supply density per region.** Home/pages/map must feel full. Stay Leiden-first until dense; scraper carries supply. Don't expand geography early.
- **Incumbents.** Hoplr (NL/BE), Nebenan (DE) own social-first hyperlocal. Our wedge = commerce-first + deals + reels, business→consumer, not neighbor chatter. Stay differentiated.
- **Online dilutes local feel.** Adding online-only webshops risks Home/pages becoming a generic deals aggregator (vs "near me"). Mitigation: scope filter defaults to surface local prominently; online gated behind follow/relevance, not spammed by distance-less volume. Online ≠ excuse to drop the geo/Leiden-first discipline for local supply.
- **Offline-only shops** = real supply gap (nothing to scrape) → need dead-simple self-serve (Phase 2).
- **Reels infra cost + moderation.** Video storage/transcode/CDN + content moderation. Defer to Phase 3.
- **GDPR.** Consumer accounts + location + follows = personal data. Consent + handling required. (EU regulation = why US apps stall here; handle deliberately.)
- **Push fatigue / store opt-in.** Conservative defaults; respect quiet hours.
- **Auth choice (open).** Passwordless email vs social login. Recommend passwordless email (reuses existing email infra) + optional Apple/Google sign-in for store norms.
- **Two-platform overhead.** Expo + EAS to keep one codebase + OTA updates.
- **App-vs-web.** Keep the web storefront for SEO/GEO (organic acquisition); app for retention/habit. They share the same backend. Don't kill web — it's the top-of-funnel.

---

## 13. Recommendation

Build **Phase 1 as an Expo React Native app over the existing Cheaply backend**: home + dedicated pages + map + profile + follow + push, supply auto-filled by the scraper/connectors so there is no empty-room cold start. Keep the web storefront alive as the SEO acquisition funnel. Layer business self-serve (Phase 2) and reels + paid boost (Phase 3) once the consumer retention loop is proven in the Leiden region.
