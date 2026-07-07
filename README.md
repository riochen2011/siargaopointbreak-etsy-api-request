# Siargao Point Break: Etsy API Access Request

![Etsy API v3](https://img.shields.io/badge/Etsy%20Open%20API-v3-F1641E?style=for-the-badge&logo=etsy&logoColor=white)
![OAuth 2.0 + PKCE](https://img.shields.io/badge/OAuth%202.0-PKCE-0E7C86?style=for-the-badge&logo=auth0&logoColor=white)
![Private seller tool](https://img.shields.io/badge/App%20type-Private%20seller%20tool-0C2233?style=for-the-badge)
![Single user](https://img.shields.io/badge/Users-1%20(the%20shop%20owner)-C99B3F?style=for-the-badge)

**Live visual version of this request:** https://riochen2011.github.io/siargaopointbreak-etsy-api-request/

**The shop:** [siargaopointbreak.etsy.com](https://siargaopointbreak.etsy.com), a single owner Etsy shop in the Philippines selling ocean inspired beaded jewelry (first line: sharktooth necklaces).

This repository documents **why** the shop is requesting Etsy Open API v3 access. It is a private, personal seller tool with exactly one user: the shop owner. It does three things: moves the owner's own Shopify listings onto Etsy, automates listing drafts with a human review step, and turns live Etsy listings into Pinterest pins and Instagram posts that link shoppers back to Etsy.

---

## The whole picture

```mermaid
flowchart LR
    subgraph OWNED["My own product sources"]
        SH["Shopify store<br/>(same products, my store)"]
        AS["Product assets<br/>(photos, prices, copy)"]
    end

    TOOL["Private listing tool<br/>runs on my computer<br/>single user: me"]

    ETSY["Etsy shop<br/><b>siargaopointbreak</b><br/>store of record"]

    subgraph SOCIAL["Marketing channels"]
        PIN["Pinterest pins"]
        IG["Instagram posts"]
    end

    BUY(["Shoppers click through<br/>and buy on Etsy"])

    SH -- "1 read my own products" --> TOOL
    AS -- "2 photos, prices, copy" --> TOOL
    TOOL -- "3 createDraftListing<br/>uploadListingImage" --> ETSY
    ETSY -- "4 getListing (read only)" --> PIN
    ETSY -- "4 getListing (read only)" --> IG
    PIN -- "5" --> BUY
    IG -- "5" --> BUY
    BUY -. "traffic returns to Etsy" .-> ETSY

    style SH fill:#5E8E3E,color:#fff,stroke:#4a7231
    style AS fill:#C99B3F,color:#fff,stroke:#a67e2e
    style TOOL fill:#0C2233,color:#fff,stroke:#0C2233
    style ETSY fill:#F1641E,color:#fff,stroke:#c94f13
    style PIN fill:#E60023,color:#fff,stroke:#b8001c
    style IG fill:#EE2A7B,color:#fff,stroke:#c11760
    style BUY fill:#E8563A,color:#fff,stroke:#c04227
```

Everything runs on the owner's computer against the owner's own accounts. Etsy stays the store of record: listings, prices, stock, and orders all live on Etsy.

---

## Use case 1: move my Shopify listings onto Etsy

I already sell the same handmade pieces on my own Shopify store. Instead of retyping every product into Etsy by hand, the tool copies my own records across and prepares Etsy **drafts**.

```mermaid
flowchart LR
    A["Read my own<br/>Shopify products"] --> B["Map fields to Etsy format<br/>title, price, tags, stock"]
    B --> C["createDraftListing<br/>uploadListingImage"]
    C --> D["I review every draft<br/>then publish by hand"]

    style A fill:#5E8E3E,color:#fff,stroke:#4a7231
    style B fill:#0E7C86,color:#fff,stroke:#0a5b64
    style C fill:#F1641E,color:#fff,stroke:#c94f13
    style D fill:#0C2233,color:#fff,stroke:#0C2233
```

## Use case 2: automate new listings and keep them in sync

New pieces start as photos and notes on my computer. The tool turns prepared assets into drafts, and keeps price and quantity on Etsy matching my records. Nothing is ever published automatically.

```mermaid
flowchart LR
    A["Prepared photos<br/>and copy"] --> B["Draft created<br/>via the API"]
    B --> C["Price and stock sync<br/>updateListing<br/>updateListingInventory"]
    C --> D["Human review before<br/>anything goes live"]

    style A fill:#C99B3F,color:#fff,stroke:#a67e2e
    style B fill:#F1641E,color:#fff,stroke:#c94f13
    style C fill:#0E7C86,color:#fff,stroke:#0a5b64
    style D fill:#0C2233,color:#fff,stroke:#0C2233
```

## Use case 3: turn listings into Pinterest pins and Instagram posts

Marketing **reads** Etsy, it never writes. The tool pulls one live listing and builds a pin and a post around it. Every piece of content links straight back to that Etsy listing, so the traffic lands on Etsy.

```mermaid
flowchart LR
    A["Read one live listing<br/>getListing + getListingImages"] --> B["Compose pin and post<br/>photo, caption, listing URL"]
    B --> C["Publish via Pinterest and<br/>Instagram official APIs"]
    C --> D(["Shoppers click back<br/>to the Etsy listing"])

    style A fill:#F1641E,color:#fff,stroke:#c94f13
    style B fill:#0E7C86,color:#fff,stroke:#0a5b64
    style C fill:#E60023,color:#fff,stroke:#b8001c
    style D fill:#E8563A,color:#fff,stroke:#c04227
```

---

## Exactly what is requested

### OAuth scopes

| Scope | Type | Why |
|---|---|---|
| `listings_r` | read | Read my listings so sync and marketing match what is live on Etsy |
| `listings_w` | write | Create draft listings, upload images, update price and stock |
| `shops_r` | read | Read my own shop profile to confirm the tool points at the right shop |

**Not requested:** transactions, billing, address, cart, favorites, feedback, or any scope that touches buyers or other sellers.

### Endpoints the tool calls

| Endpoint | Purpose | Use case |
|---|---|---|
| `getShop` | Confirm shop identity at startup | 1, 2, 3 |
| `getListingsByShop` | List my listings for sync and content planning | 2, 3 |
| `createDraftListing` | Create a draft, never a live listing | 1, 2 |
| `uploadListingImage` | Attach my product photos to a draft | 1, 2 |
| `updateListing` | Apply corrections I approve | 2 |
| `updateListingInventory` | Keep price and quantity matching my records | 2 |
| `getListing` | Read one listing to build a pin or post | 3 |
| `getListingImages` | Fetch the listing photo for that pin or post | 3 |

### How it behaves

- **Low volume.** A short batch run when I add products and a small daily sync. Dozens of calls a day, far under default rate limits.
- **OAuth 2.0 with PKCE.** Tokens are stored in a local file on my computer, never committed, never shared.
- **Backoff built in.** The tool respects rate limit headers and retries with exponential backoff.
- **Nothing auto publishes.** The API writes drafts and sync updates. Publishing is always a manual decision.

---

## Responsible use

| It does | It never does |
|---|---|
| Works only on siargaopointbreak, a shop I own | No scraping or crawling of Etsy pages or other shops |
| Creates drafts for my review | No access to buyer data, other sellers, or marketplace analytics |
| Keeps Etsy price and stock accurate | No resale or sharing of Etsy data with any third party |
| Sends social shoppers to Etsy | No automated reviews, messages, or purchases |
| Follows the Etsy API Terms of Use | No public distribution, no other users, keys stay private |

---

The term "Etsy" is a trademark of Etsy, Inc. This application uses the Etsy API but is not endorsed or certified by Etsy, Inc. Shopify, Pinterest, and Instagram are trademarks of their respective owners, mentioned here only to describe the workflow.
