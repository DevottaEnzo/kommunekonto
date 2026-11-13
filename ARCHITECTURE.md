# Kommunekonto – how it works

A local proof of concept that lets a Norwegian kommune:

1. pick itself from the official register (Brreg),
2. prove that the person is a kommune employee (a code sent to their work email),
3. choose a handle, ideally on the kommune's own domain,
4. get an AT Protocol account created for it,
5. connect its news feed, which a bot then posts to the account automatically.

Everything runs on one machine against the `local-sandbox` PDS. This document
explains what the pieces are and how they work together. The code itself is
not published.

The wizard's text is in Nynorsk. This document is in English.

---

## 1. The big picture

```mermaid
flowchart LR
  user(["Kommune employee<br/>in a browser"])

  subgraph app["Kommunekonto app (app/)"]
    ui["Wizard UI<br/>React + Vite<br/>app/src"]
    api["Node server<br/>app/server"]
    sched["Feed scheduler<br/>every 10 min"]
    store[("Local storage<br/>app/.runtime")]
  end

  subgraph sandbox["local-sandbox (Docker)"]
    pds["PDS<br/>127.0.0.1:2583"]
    plc["PLC directory<br/>127.0.0.1:2582"]
  end

  subgraph web["Internet"]
    brreg["Brreg<br/>Enhetsregisteret"]
    site["Kommune website<br/>RSS / Atom feed"]
    dns["Public DNS<br/>_atproto TXT"]
  end

  user --> ui
  ui -- "/api/*" --> api
  sched --> api
  api --- store
  api -- "XRPC" --> pds
  pds --> plc
  api --> web
```

| Piece | What it does |
| --- | --- |
| **Wizard UI** | Five steps in one page. Talks only to its own server. |
| **Node server** | All rules and all outside calls. Holds the passwords. |
| **Scheduler** | Runs inside the server and syncs every connected feed. |
| **PDS** | Where the accounts and posts live. In the sandbox it has no internet access. |
| **Brreg** | Source of the 357 kommuner, their domains and their written language (Nynorsk or Bokmål). |
| **Kommune website** | Where the news comes from. |
| **DNS** | Proves that a kommune controls a domain before the account's handle moves there. |

---

## 2. The wizard, step by step

```mermaid
stateDiagram-v2
  [*] --> Kommune
  Kommune --> Epost: kommune chosen
  Epost --> Brukarnamn: code confirmed, no account yet
  Epost --> Konto: code confirmed, kommune already has an account
  Brukarnamn --> Konto: handle chosen
  Konto --> Nyheiter: account exists
  Nyheiter --> [*]: feed connected

  Kommune: 1 · Kommune
  Kommune: search Brreg by name or number
  Epost: 2 · E-post
  Epost: 6-digit code to a kommune address
  Brukarnamn: 3 · Brukarnamn
  Brukarnamn: service handle + optional kommune domain
  Konto: 4 · Konto
  Konto: create account, profile, bot app password
  Nyheiter: 5 · Nyheiter
  Nyheiter: find or paste feed, first post, scheduler
```

Every step after the email step needs a **verified cookie**. The server reads the
kommune from that cookie, never from what the browser sends, so nobody can act
for a kommune they have not verified for.

### Step 1 – Kommune

```mermaid
sequenceDiagram
  participant B as Browser
  participant S as Server
  participant C as kommuner.json cache
  participant R as Brreg

  B->>S: GET /api/kommuner?q=tromso
  S->>C: fresh (under 24 h)?
  alt cache is stale or missing
    S->>R: enheter?organisasjonsform=KOMM&size=1000
    R-->>S: 357 kommuner
    S->>C: save
  else Brreg is down
    S->>C: use last cache
  end
  S-->>B: up to 10 matches, æøå-insensitive
```

For each kommune the server works out:

- **Allowed email domains:** the homepage host without `www.`, plus the domain of
  Brreg's email address. 354 of 357 have at least one.
- **Post language:** `nn` for Nynorsk and `nb` for Bokmål, from Brreg's `maalform`.

`TEST_KOMMUNE_DOMAINS=devotta.no` adds a fake **Testkommune** (orgnr 999999990)
for testing.

### Step 2 – Email code

```mermaid
sequenceDiagram
  participant B as Browser
  participant S as Server
  participant M as Mailer (dev: page + terminal)

  B->>S: POST /api/verify/start {orgnr, email}
  S->>S: is the email on the kommune's domain?
  S->>S: rate limits: 30 s gap, 5 per hour per address, 20 per hour per IP
  S->>S: new 6-digit code, stored only as an HMAC, valid 10 min
  S->>M: send code
  S-->>B: challengeId
  B->>S: POST /api/verify/confirm {challengeId, code}
  alt right code (max 5 tries)
    S-->>B: Set-Cookie kommune_verified<br/>HMAC-signed, HttpOnly, SameSite=Strict, 1 h
  else wrong or expired
    S-->>B: error, tries left
  end
```

There is **no real email sending yet**. In `pnpm dev` the mail is printed in
the terminal and shown in a "Prototype-innboks" box on the page.

### Step 3 – Handle

```mermaid
flowchart TD
  start(["Kommune: Alver"]) --> sug["Suggestions from the name:<br/>alver, alver-kommune, alverkommune, alver-nyheiter"]
  sug --> pick["User picks or types a name"]
  pick --> rules{"One label, a–z 0–9 and hyphen,<br/>3–18 chars, not reserved?"}
  rules -- no --> pick
  rules -- yes --> free{"resolveHandle on the PDS:<br/>is name.test free?"}
  free -- taken --> pick
  free -- free --> svc["Service handle: alver.test"]

  svc --> dom{"Also want a kommune domain?<br/>e.g. nyheiter.alver.kommune.no"}
  dom -- no --> done(["Go to account step"])
  dom -- yes --> own{"Domain is one of the kommune's<br/>Brreg domains or a subdomain?"}
  own -- no --> dom
  own -- yes --> txt{"_atproto TXT record already there?"}
  txt -- yes --> pub(["Kommune probably already has an account:<br/>suggest Eurosky Publisher instead"])
  txt -- no --> done
```

The account always starts on the service handle. The kommune domain is saved
and switched to later (see section 4), because the DNS record has to contain
the account's DID, which only exists after the account is created.

### Step 4 – Create the account

A "pending" record is written **before** the PDS call. If the PDS gives an unclear
answer, the attempt is not lost or repeated by accident.

```mermaid
flowchart TD
  a(["POST /api/account {handle, customDomain}"]) --> chk{"Kommune already has<br/>an account or an unchecked attempt?"}
  chk -- yes --> stop1(["409: stop"])
  chk -- no --> pw["Random 48-byte password"]
  pw --> pend[("Save PENDING record<br/>encrypted account store")]
  pend --> create["PDS createAccount<br/>handle, password, verified work email"]
  create --> ok{"Result?"}
  ok -- "handle or email refused" --> clean["Delete pending"] --> stop2(["409: choose again"])
  ok -- "unclear / network error" --> keep(["502: keep pending,<br/>block retries until checked by hand"])
  ok -- "account created" --> cred[("Save password<br/>encrypted")]
  cred --> app["createAppPassword 'kommune-bot'"]
  app --> cred2[("Save bot app password")]
  cred2 --> acc[("Save public account facts<br/>delete pending")]
  acc --> prof["Write profile:<br/>display name + description in nn/nb"]
  prof --> fin(["Account ready"])
```

Two choices worth knowing:

- The account gets the **verified work email**. The kommune can then reset the
  password through the PDS, so it is never locked to this service.
- The bot gets its own **app password**. The main password is only used to
  change the handle.

### Step 5 – Connect the news feed

```mermaid
flowchart TD
  s(["Nyheiter step"]) --> disc["Auto-discover:<br/>Brreg homepage, then each email domain,<br/>with and without www."]
  disc --> found{"Feed found?"}
  found -- no --> paste["User pastes the feed URL"]
  found -- yes --> prev
  paste --> guard{"Public address?<br/>(no localhost, private or link-local IPs)"}
  guard -- no --> paste
  guard -- yes --> prev["Preview: 5 newest items"]
  prev --> go["Start publisering"]
  go --> save[("Save feed for the kommune<br/>feeds.sqlite")]
  save --> first["First sync now:<br/>post only the newest item"]
  first --> on(["Scheduler takes over"])
```

The step also has "Sjekk feeden no" (run now), pause/resume, the last run's
result, the total posted, and the latest posts read back from the PDS.

---

## 3. The bot

### One sync

The same `syncFeed` runs from the wizard, the scheduler and the `pnpm feed` CLI.

```mermaid
flowchart TD
  a(["syncFeed"]) --> f["Fetch RSS or Atom<br/>size and time limits"]
  f --> n["Drop items already handled<br/>(match on id OR url)"]
  n --> first{"First sync of this feed<br/>for this account?"}
  first -- yes --> skip["Keep newest 1 item<br/>mark the rest 'skipped'<br/>(never post the archive)"]
  first -- no --> all["All unseen items"]
  skip --> cap
  all --> cap["At most 5 per run<br/>oldest first"]
  cap --> loop{"Next item"}
  loop -- none left --> done(["Return posted / skipped / pending"])
  loop --> meta["Need image or summary?<br/>read og:image / og:description from the article"]
  meta --> thumb["Upload image as blob<br/>(max 976 KB, jpeg/png/webp/gif)"]
  thumb --> post["createRecord app.bsky.feed.post<br/>title + link card + langs"]
  post --> mark[("Mark item 'posted' with its URI")]
  mark --> loop
```

A missing or too-large image only means the link card has no picture. It never
stops the post.

### What a post looks like

```mermaid
flowchart LR
  item["Feed item<br/>title, link, summary, image"] --> rec
  subgraph rec["app.bsky.feed.post"]
    t["text: title (max 300 graphemes)"]
    l["langs: nn or nb"]
    subgraph emb["embed: app.bsky.embed.external"]
      u["uri: article link"]
      ti["title"]
      d["description: summary"]
      th["thumb: uploaded image blob"]
    end
  end
```

### Scheduler and runner

```mermaid
sequenceDiagram
  participant T as Timer (every 10 min)
  participant R as FeedRunner
  participant D as feeds.sqlite
  participant A as Account store
  participant P as PDS

  T->>R: runAll()
  R->>D: enabled feeds
  loop each kommune, one after another
    R->>A: botLogin(orgnr): app password
    R->>P: createSession (app password)
    R->>R: session DID == feed's account DID?
    R->>R: syncFeed(...)
    R->>D: record run: ok/failed, message, posted count
  end
  Note over R: One kommune failing does not stop the others.<br/>Two runs for the same kommune at once share one sync.
```

Settings: `FEED_INTERVAL_MINUTES` (default 10), `FEED_SCHEDULER=off` to turn it off.

---

## 4. Moving the handle to the kommune's domain

This can't be tested in the sandbox, because the sandbox PDS has no DNS and its
DIDs only exist in the local PLC. It needs the shared hackathon PDS.

```mermaid
sequenceDiagram
  participant K as Kommune IT
  participant B as Wizard (Konto step)
  participant S as Server
  participant D as Public DNS
  participant P as PDS

  S-->>B: Add TXT _atproto.nyheiter.alver.kommune.no = "did=did:plc:…"
  K->>D: add the TXT record
  B->>S: POST /api/account/switch-handle
  S->>D: look up _atproto TXT
  alt record matches the account's DID
    S->>P: createSession (main password)
    S->>P: updateHandle nyheiter.alver.kommune.no
    P->>D: PDS checks the record itself
    S-->>B: handle is now @nyheiter.alver.kommune.no
  else not there yet, or points to another DID
    S-->>B: "not found yet, try again in a few minutes"
  end
```

---

## 5. Where data is kept

All of it is under `app/.runtime/` and **not** in git. This is local
prototype storage, not a production design.

```mermaid
flowchart LR
  subgraph rt["app/.runtime/"]
    k[("kommuner.json<br/>Brreg cache, 24 h")]
    subgraph acc["accounts/ (encrypted, AES-256-GCM)"]
      a1["kommune-account<br/>orgnr → DID, handle, domain, who created it"]
      a2["kommune-credential<br/>DID → main password, bot app password"]
      a3["kommune-pending<br/>orgnr → unfinished attempt"]
      key["encryption.key<br/>(or AUTH_ENCRYPTION_KEY)"]
    end
    subgraph fd["feeds.sqlite"]
      f1["kommune_feeds<br/>feed URL, language, enabled, last run"]
      f2["feed_items<br/>every item posted or skipped"]
    end
  end
  mem["Server memory<br/>pending email codes, rate limits"]
  pds[("Sandbox PDS<br/>accounts, profiles, posts")]
```

| Data | Lives in | Lost when |
| --- | --- | --- |
| Email codes, rate limits | Server memory | The server restarts |
| Verified cookie | Browser, signed with `VERIFY_SECRET` | After 1 h, or on restart if no secret is set |
| Passwords | `accounts/`, encrypted | The folder or key is deleted |
| What was posted | `feeds.sqlite` | The file is deleted |
| Accounts and posts | Sandbox PDS | The sandbox is stopped or restarted |

---

## 6. Code map

```mermaid
flowchart TD
  subgraph src["app/src (browser)"]
    main["main.tsx<br/>steps + navigation"]
    main --> s1["KommuneStep"]
    main --> s2["EmailStep"]
    main --> s3["HandleStep"]
    main --> s4["AccountStep"]
    main --> s5["FeedStep"]
  end

  subgraph srv["app/server"]
    entry["dev.ts / index.ts"] --> setup["setup.ts<br/>wires services from .env"]
    setup --> appts["app.ts<br/>HTTP routes, security headers"]
    appts --> kom["kommuner/brreg.ts"]
    appts --> ver["verify/<br/>codes, cookie, mailer"]
    appts --> han["handles/<br/>name rules, availability, DNS check"]
    appts --> accm["accounts/<br/>create, bot login, handle switch"]
    appts --> feedm["feed/routes.ts"]
    feedm --> run["feed/runner.ts<br/>scheduler"]
    run --> pub["feed/publish.ts<br/>syncFeed"]
    pub --> parse["feed/feed.ts<br/>parse + discover"]
    pub --> at["feed/atproto.ts<br/>XRPC client"]
    pub --> st["feed/store.ts"]
    feedm --> guard["feed/guard.ts<br/>public-URL check"]
    accm --> at
    accm --> auth["Encrypted<br/>account store"]
    cli["feed/cli.ts<br/>pnpm feed"] --> pub
  end

  src -- "fetch /api/*" --> appts
```

Reused from Demokratiløftet:

| Repo | How it is used |
| --- | --- |
| `local-sandbox` | The PDS and PLC everything runs against |
| `app-protospace` | The app template (React, Vite, Node server, checks) |
| `organisation-verifier` | Reference for the Brreg lookups |

---

## 7. API

All `POST` requests must come from the app's own origin. All routes except
`/api/kommuner*` need the verified cookie.

| Method | Path | What it does |
| --- | --- | --- |
| GET | `/api/kommuner?q=` | Search kommuner (max 10) |
| GET | `/api/kommuner/<orgnr>` | One kommune |
| POST | `/api/verify/start` | Send a code `{orgnr, email}` |
| POST | `/api/verify/confirm` | Check the code `{challengeId, code}`, sets the cookie |
| GET | `/api/verify/status` | Who is verified, if anyone |
| POST | `/api/verify/reset` | Forget the verification |
| GET | `/api/handle/options` | Name suggestions and the service domain |
| GET | `/api/handle/check?name=&domain=` | Is this handle valid and free? |
| GET | `/api/handle/custom?domain=` | Is this kommune domain allowed and unused? |
| GET | `/api/account` | The kommune's account and DNS status |
| POST | `/api/account` | Create the account `{handle, customDomain?}` |
| POST | `/api/account/switch-handle` | Move to the kommune domain once DNS is ready |
| GET | `/api/feed` | Connected feed, last run, latest posts |
| GET | `/api/feed/discover` | Look for a feed on the kommune's domains |
| GET | `/api/feed/preview?url=` | 5 newest items of a feed |
| POST | `/api/feed` | Connect a feed `{url}` and run the first sync |
| POST | `/api/feed/run` | Sync now |
| POST | `/api/feed/enabled` | Pause or resume `{enabled}` |

---

## 8. Security measures

| Risk | Measure |
| --- | --- |
| Someone acts for a kommune they don't work for | Email code to the kommune's own domain. The server takes the kommune from the signed cookie, not from the request |
| Guessing codes | 6 digits, 5 tries, 10 minutes, stored only as an HMAC |
| Spamming codes | 30 s between sends; 5 per hour per address; 20 per hour per IP |
| Cross-site requests | Cookie is `HttpOnly` and `SameSite=Strict`; every `POST` is checked for the app's own `Origin` |
| Stolen passwords at rest | Encrypted with AES-256-GCM |
| A leaked bot password | The bot only has an app password, which can be revoked without touching the account |
| The kommune locked out | Account uses their work email, so they can reset the password at the PDS |
| Feed URL pointing inside the network (SSRF) | Only public addresses are allowed; localhost, private and link-local IPs are refused |
| Posting a whole archive at once | First sync posts 1 item; later runs at most 5 |
| Page injection | Strict Content-Security-Policy, `nosniff`, no referrer, no framing |

---

## 9. What is still missing

- Real email sending instead of the dev mailer.
- Testing on the shared hackathon PDS: the kommune-domain handle switch, and
  seeing posts in a real Bluesky or Eurosky app (the sandbox has no AppView).
- A real storage design (codes are in memory; SQLite files under `.runtime/`).
- A manual approval path for kommuner with no domain in Brreg (Loppa, Lyngdal,
  Ullensvang) and for account attempts that ended without a clear answer.
- A news-only feed or a filter for CMSs that mix news with service pages
  (Alver's Prokom site).
- An SSRF check that also follows redirects (today only the first address is checked).
- Only 1 of 11 kommuner tried (Alver) had a usable feed, so most kommuner
  will need to paste a feed URL or get one set up.
