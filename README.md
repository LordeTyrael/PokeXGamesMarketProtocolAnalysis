# Market Protocol — Technical Documentation (PxG)

Technical analysis and reverse engineering notes for the **Market** system of the PokeXGames client (`pxgme.exe`).

---

## 1. How the Market Talks

There is no HTTP endpoint and no dedicated packet opcode for the market. It rides on the game's **extended opcode** channel over the main connection:

* The market's extended opcode is **253** (`AdditionalExtendedIds.Market`).
* On the wire, packets are written as opcode `0x32`, then one byte (`253`), then the serialized payload.
* Incoming payloads land in `ProtocolGame::onExtendedOpcode` (RVA `0x2949B0`).
* Outgoing payloads leave through `ProtocolGame::sendExtendedOpcode` (RVA `0x27CE60`).

That's the whole transport. Everything else — compression, encoding, actions — lives inside the payload.

## 2. Inside the Payload

### 2.1. zlib Compression

Raw payloads are zlib-compressed, with a small size prefix in front so the client knows how much to allocate:

| Prefix Byte | Meaning |
| :--- | :--- |
| `0x01` | uncompressed length is 1 byte, compressed length is 2 bytes (little-endian) |
| `0x02` | uncompressed length is 2 bytes, compressed length is 2 bytes (little-endian) |

After the prefix comes the zlib stream. Inflate it and you get a custom Lua-table serialization — not JSON, not protobuf.

One thing worth knowing if you're *sending* instead of receiving: `sendExtendedOpcode` expects the **uncompressed** buffer. Compression happens later, inside the C++ network path.

### 2.2. Lua-Table Serialization

The decompressed bytes are a recursive type-tagged structure:

| Type Byte | Encoding |
| :--- | :--- |
| `0x00` | `nil` |
| `0x01` | boolean: 1 byte (`0`/`1`) |
| `0x02` | signed 8-bit integer |
| `0x03` | signed 16-bit integer (LE) |
| `0x04` | signed 32-bit integer (LE) |
| `0x05` | signed 64-bit integer (LE) |
| `0x06` | double: 8 bytes IEEE-754 |
| `0x07` | string: `u16` length LE + UTF-8 bytes |
| `0x08` | table: `u16` entry count LE + `(key, value)` pairs |

Every server-to-client response is wrapped in a top-level root table whose first entry identifies the action:

```
{ 1: { action: "actionName", ... } }
```

## 3. Market Actions

### 3.1. Server → Client

Six actions arrive from the server:

* **`open`** — pushed when the market window opens. Carries everything the UI needs to bootstrap: categories, validDurations, pokemonTiers, buyRequestCategories, and levelRequiredToMakeAnOffer.
* **`refreshSells`** — active global market listings.
* **`refreshMySells`** / **`refreshMyOffers`** — the local player's own listings and bids.
* **`refreshBuyRequests`** / **`refreshMyBuyRequests`** — public buy requests, and the player's own.
* **`refreshSell`** — the response to `getSell(sellId)`; this is what right-click → "show offers" triggers for a single listing.

A real `refreshSells`, captured live:

```json
{
  "pages": 144,
  "startIndex": 0,
  "sells": {
    "1": {
      "price": 100,
      "item": {
        "id": 1234,
        "count": 1,
        "name": "seed",
        "description": "You see 1 seed."
      },
      "id": 15173055,
      "description": "",
      "time": 1782178931,
      "remaining": 345599,
      "playerName": "SellerName"
    }
  }
}
```

(`sells` keys are contiguous integers, so the same table may surface as either a JSON object or array depending on the decoder.)

**Reading the listing fields:**

* **price** — cents (Int32). This is the **unit price**: a purchase costs `price × count` total (`buy_now.otui` confirms: `Total = data.price * count`). Listings with `price = 0` are **offer-only** entries — the game shows `-` as unit price. They're common (~15% of listings) and always sort last on price sorts; other sorts ignore them.
* **item** — nested table: `id`, `count`, `name`, `description`.
* **id** — unique listing ID (Int32).
* **description** — the seller's custom note.
* **time** — Unix timestamp of listing creation (Int32).
* **remaining** — seconds until expiry (Int32).
* **playerName** — seller character name.

### 3.2. Client → Server

Outgoing arguments are serialized per-argument and concatenated in order:

| Action | Serialized Arguments |
| :--- | :--- |
| `getSells` | `"getSells"`, category, page, column, ascending, search |
| `getBuyRequests` | `"getBuyRequests"`, category, page, column, ascending, search |
| `getMyBuyRequests` | `"getMyBuyRequests"` |
| `getMySells` | `"getMySells"` |
| `getMyOffers` | `"getMyOffers"` |
| `getSell` | `"getSell"`, sellId |
| `buyNow` | `"buyNow"`, sellId, count |

On the wire that looks like `getSells("All", 0, "time", false, "")` — `"All"` and `"time"` are literal strings, and **pages are 0-based** (UI page 1 sends `0`).

#### The Refresh Bundle

The Refresh button does not send `getSells` alone. It fires a bundle of all five `get*` actions at once — and the server needs that first bundle: before it, a lone `getSells` gets no response. After the first bundle has been processed, though, lone page requests work fine — that's exactly what the next-page button does. So: bundle once, then request pages individually.

#### The Session Gate

Bigger gotcha: **the server ignores every market action until the market window has been opened once per game session.** And `open` isn't requestable — it's server-pushed by an actual in-game interaction (talking to the market NPC/PC). There is no known opcode to trigger it programmatically. Any automation has to have the window opened manually after login, or it will send perfectly-formed packets into silence.

## 4. Client Internals (pxgme.exe)

### 4.1. Key Functions

| Function | RVA | Image Address | Notes |
| :--- | :--- | :--- | :--- |
| `ProtocolGame::sendExtendedOpcode` | `0x27CE60` | `0x14027CE60` | Sends extended opcode packets. Aborts if ProtocolGame + 0x41 is non-zero. |
| `ProtocolGame::onExtendedOpcode` | `0x2949B0` | `0x1402949B0` | Handles incoming extended opcodes. Sets ProtocolGame + 0x41 = 1 when the extended opcode byte is 0. |
| `ProtocolGame::parseMessage` | `0x29B060` | `0x14029B060` | Main incoming protocol dispatcher. |
| Jump table (parseMessage) | `0xF5D7C0` | `0x140F5D7C0` | 32-bit relative jump table for opcodes ≥ `0x0A`. |

### 4.2. Global Pointers

The pointer you actually want is the active `ProtocolGame*` instance, sitting in `.bss` at RVA `0x1313EF8` (`0x141313EF8`). The game keeps it updated, and it's the same instance used by manual refreshes.

The famous `g_game` singleton (RVA `0x10EA560`) looks tempting but is a dead end: static analysis shows the memory there is a hash-map-like structure, not a plain singleton with a `ProtocolGame*` member at a simple offset. The old `Game + 0x18` trick is invalid in the current client — using it as `this` for `sendExtendedOpcode` produces an access violation.

### 4.3. The Send-Lockout Flag

Byte `+0x41` inside `ProtocolGame` is a lockout flag. If it's non-zero, `sendExtendedOpcode` silently aborts and logs an internal error instead of transmitting. `onExtendedOpcode` sets it to 1 when the incoming extended opcode byte is 0 — so after certain traffic, your outgoing sends do nothing until the flag is cleared. Tools that inject sends need to zero the byte before each call and restore it afterward.

### 4.4. Buffer Layouts

**InputMessage (read side):**

| Offset | Type | Field |
| :--- | :--- | :--- |
| `+0x1C` | Int32 | `m_bufStart` |
| `+0x20` | Int32 | `m_readPos` |
| `+0x24` | Int32 | `m_bufEnd` |
| `+0x28` | Byte[] | data buffer |

Reading a byte: `inputMsg.add(0x28).add(readPos).readU8()`.

**OutputMessage (write side):**

| Offset | Type | Field |
| :--- | :--- | :--- |
| `+0x20` | Int32 | `m_writePos` |
| `+0x24` | Int32 | `m_checksumPos` |
| `+0x28` | Int32 | `m_packetSizePos` |
| `+0x2C` | Byte[] | data buffer |

For building arguments manually: `std::string` uses the libstdc++ cxx11 ABI — `[dataPointer 8][size 8][capacity/localBuf 16][padding 8]`, with SSO storing data at offset `+16` for strings ≤ 15 bytes.

### 4.5. Gold and Currency

The client makes reading the player's gold surprisingly hard:

* The standard Tibia/OTClient resource-balance opcode (`GameServerResourceBalance`, `0xEE`) is **not handled** by this build. In the dispatcher's jump table, opcodes `0xEE`, `0xEF`, and `0xF2` all point to the default unhandled case.
* Market payloads never carry a balance field — captured `open` and `refreshSells` payloads contain no `gold`/`balance` anywhere.
* Premium currency (Diamonds) goes through dedicated Lua bindings (`getDiamonds()`); standard Gold validation happens entirely server-side when market operations execute.

Net result: there is no fixed memory location holding the player's gold, and no packet announcing it. Anything that needs the balance has to track it externally.

## 5. Pokémon Attributes in Listings

Pokémon sold in Pokéballs carry their full details embedded directly in the item's `description` string — there's no structured field for them. The description text packs everything:

* **Ball type** — e.g., `Ultra Ball`, `Yume Ball`
* **Name + boost** — species name with enhancement suffix, e.g., `Charizard +70`
* **Enhancement** — the numeric `+N` value
* **Addons** — cosmetic count, e.g., `addons=3`
* **Held items** — name and tier, e.g., `X-Lucky (Tier: 7)`
* **Required level** — e.g., `lvl=100`

Parsing it is pure string work against the description field — a handful of anchored regexes over the patterns above recovers every attribute.
