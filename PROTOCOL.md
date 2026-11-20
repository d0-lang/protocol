# The d0 lookup protocol, v1

Every endpoint path starts with the protocol version, e.g. `/v1/get-key`. Bodies are JSON. The examples below use `//` comments for explanation; real bodies have none.

Clients ignore fields they don't know, and servers ignore query parameters and request body fields they don't know. Within `v1`, new optional fields, new query parameters and new `features` strings may be added; anything else is a new version.

## Server info (outside the lookup loop)

### Server key

Each server has an Ed25519 key pair. Clients use its public key to check that metadata really comes from that server.

GET `/v1/get-key` serves the public key. This response is not signed. The client stores the key and checks later metadata against it.

```jsonc
{ "key": "<base64 Ed25519 public key>" }
```

### Metadata

GET `/v1/get-meta` serves metadata. The client sends an `X-Nonce: <string>` header, and the server copies it into the `nonce` field. The client rejects a response whose `nonce` is not the one it sent, and keeps the data it has stored.

The client fetches metadata when the user adds the server, each time the client starts, and at least once a day while it runs.

The response is signed. `X-Signature` is a base64 Ed25519 signature over the exact body bytes sent. Neither side re-serializes the JSON to sign or verify it.

```jsonc
{
  "name": "my server",              // display name
  "nonce": "...",                   // the X-Nonce the client sent
  "languages": ["de_en", "es_en"],  // language pairs served
  "features": ["context", "context-match", "labels"], // optional; missing means []
  "verification_urls": ["https://..."],     // pages where maintainers post server news
  "sources": [                      // optional, see Sources
    {
      "name": "Wiktionary",
      "url": "https://en.wiktionary.org/",
      "license": "CC-BY-SA-4.0",    // optional
      "license_url": "https://creativecommons.org/licenses/by-sa/4.0/", // optional
      "languages": ["de_en"]        // optional; missing means every pair
    },
    { "name": "Wikimedia Commons", "url": "https://commons.wikimedia.org/" }
  ],
  "report_url": "https://...",      // optional, see Reports
  "legal": {                        // optional, see Legal
    "privacy_link": "https://...",
    "privacy_hash": "<sha256 hex>",
    "privacy_change": "...",        // optional
    "tos_link": "https://...",
    "tos_hash": "<sha256 hex>",
    "tos_change": "..."             // optional
  }
}
```

### Identity change

If the signature does not verify with the stored key, the client fetches `/v1/get-key` again.

- Same key: the response is broken. Show an error and keep the old data.
- Different key: the server's identity changed. Show the `verification_urls` stored from the last verified metadata, not the ones in the new response, and lock the message for 5 seconds. If the user continues, store the new key and accept the new metadata only if it verifies with that key. If it does not verify, treat it as a broken response: show an error and keep the old key and data. If the user does not continue, keep the old key.

While a server has a broken response or an unaccepted identity change, the client sends it no lookups. This lasts until a fetch verifies with the stored key, or the user accepts a new key whose metadata verifies.

On the first bind there is nothing stored yet, so the client trusts the key from `/v1/get-key`.

### Domain move

When maintainers move the server to a new domain, `/v1/get-meta` returns this body instead of metadata. It is signed the same way and its `nonce` must match `X-Nonce`.

```jsonc
{ "nonce": "...", "new_source": "https://..." }
```

The signing key stays the same, so this is not an identity change and shows no warning. If the signature verifies, the client replaces the stored URL with `new_source` and fetches metadata from there with the same key. If it does not verify, the client ignores `new_source`.

### Legal

The server hosts its privacy policy and terms of service itself and links to them. The client does not show server-written policy text. It writes the disclosures itself, from what the protocol tells it:

- Always: the server sees every word looked up, and the user's IP address.
- If `features` lists `context`: the server also receives the sentence around the word.
- Then "Learn more", linking `privacy_link` and `tos_link`.

`privacy_hash` and `tos_hash` are the sha256, in lowercase hex, of the document the server publishes at the link, taken over its source text rather than the rendered page, so a layout change does not count as a change. The client treats each hash as an opaque marker: it stores it and compares it, and never fetches the link to check it. The metadata signature is what makes it trustworthy.

When the client binds a server, it stores the URL, the key, the `verification_urls` and both hashes.

When a later metadata fetch carries a hash that differs from the stored one, or adds or drops the `legal` section, the client stops sending lookups to that server until the user acknowledges the change. The prompt names the document that changed, shows `privacy_change` or `tos_change` if present, labeled as the server's own summary, and links the document. Without a change text it only says the document changed. On acknowledge, the client stores the new hash. Change texts are at most 500 characters and are not shown on the first bind.

If metadata has no `legal` section, the client tells the user to use the server only for testing, and only if they are affiliated with its developers or maintainers. The message is locked for 5 seconds; on Continue the client binds the server as usual. A server that lists `context` without `legal` gets no sentences until the user agrees to that in the same message.

### Sources

`sources` lists whose material the server's answers are built from, so the client can credit them. A server whose data needs credit (Wiktionary is CC BY-SA) lists it here.

- `name`: who is credited, as the client shows it. At most 100 characters; short names fit best (`Wiktionary`).
- `url`: the source's site, `https`. Its origin also decides which links in a lookup answer the client shows (see `sources` on entries and `source` on audio).
- `license`: optional. The SPDX identifier where one exists (`CC-BY-SA-4.0`), otherwise the licence's name. At most 100 characters. Left out when each item has its own licence (recordings on Wikimedia Commons).
- `license_url`: optional link to the licence text, `https`.
- `languages`: optional list of the language pairs the source is used for. Missing means every pair.

The client writes the credit itself: the data is derived from `name`, under `license`, with the links. It shows the list when the user binds the server and on the server's page in the settings. A changed `sources` list is not a legal change and does not pause lookups.

In a lookup answer, entries and audio carry links to the exact page they come from. The client shows such a link only when its origin is the origin of a `url` in `sources`, and names it with that source's `name`. So the server sends no credit text with a lookup, and can only link to sites named in signed metadata.

### Reports (optional)

`report_url` is the page where users report a wrong or missing answer from this server, `https`. A server without one leaves the field out, and the client then offers no report for that server's answers.

When the user asks to report a lookup, the client opens `report_url` in the browser, with these query parameters added to any the URL already has:

- `word` and `lang`: as sent in the lookup.
- `client`: the client's name and version (`dadict/0.6.0`).

```
https://example.org/report?form=word
→ https://example.org/report?form=word&word=Mutter&lang=de_en&client=dadict%2F0.6.0
```

The client adds nothing else: not the sentence around the word, and not the page the user is on. The user can write those into the report. The page may belong to someone other than the server (an issue tracker), and it learns the word, so the client opens it only when the user asks, and names the host it leads to.

A changed `report_url` is not a legal change and does not pause lookups.

### Label explanations (optional)

A server that explains its `labels` lists `"labels"` in `features` and serves GET `/v1/get-labels?lang=ru_en,de_en`.

- `lang` (required): one or more language pairs, joined by commas. The client asks for the pairs the user has in use on that server.

```jsonc
{
  "labels": {
    "ru_en": {
      "pf": {
        "name": "perfective",
        "text": "The action is seen as a whole, usually completed. Most verbs come in pairs: писать (impf) and написать (pf)."
      },
      "anim": { "name": "animate" }
    },
    "de_en": {}
  }
}
```

- `labels`: per pair, each label string the server sends for that pair, with its explanation. A pair the server does not serve is left out.
- `name`: what the label stands for, at most 60 characters.
- `text`: optional longer explanation, plain text, at most 1000 characters.
- Both are written in the pair's target language, like the glosses.

The response is not signed, like a lookup. The client fetches it when it fetches metadata (on start and at least once a day) and when the user starts using a new pair, and keeps the last copy. A failed fetch never stops lookups: a label without an explanation is shown without one.

`pos` needs no explanation from the server: it is a Universal Dependencies tag, and the client names those itself.

## The lookup loop

### Word lookups

A language pair is `<source>_<target>`: the language of the looked-up word, then the language its meaning is given in (`de_en`, `pl_en`). The two may be the same language (`de_de`): the glosses are then definitions in that language, and nothing else changes. Each is a lowercase ISO 639 code: the two-letter ISO 639-1 code by default (`de`, `ja`), and the three-letter ISO 639-3 code for a language that has no two-letter one (`yue_en` for Cantonese).

Request: POST `/v1/lookup` with a JSON body (`Content-Type: application/json`):

```jsonc
{ "word": "Mutter", "lang": "de_en" }
```

- `word` (required): the word the client's tokenizer picked out. At most 64 characters.
- `lang` (required): a language pair the server lists in metadata.
- `context` (optional): the sentence or subtitle line containing `word`, or a part of it around `word`. At most 500 characters. See Lookup context.
- `at` (optional): a number, which occurrence of `word` in `context` was clicked. See Lookup context.

Lengths are counted in UTF-16 code units. A client does not look up a longer `word`. When the sentence is longer than 500, it sends the part around `word` (see Lookup context), and without `context` when it can't. A server may answer a longer `context` with `400` `bad_request`.

Response:

```jsonc
{
  "entries": [
    {
      "lemma": "die Mutter",
      "pos": "NOUN",
      "pron": {
        "ipa": ["/ˈmʊtər/", "[ˈmʊtɐ]"],
        "audio": [
          {
            "url": "https://upload.wikimedia.org/wikipedia/commons/0/03/De-Mutter.ogg",
            "source": "https://commons.wikimedia.org/wiki/File:De-Mutter.ogg"
          }
        ]
      },
      "senses": [
        { "glosses": ["mother"] },
        { "glosses": ["nut (for a bolt)"] }
      ],
      "sources": ["https://en.wiktionary.org/wiki/Mutter#German"]
    }
  ]
}
```

Russian entry with `labels` (German puts the article into `lemma` instead):

```jsonc
{
  "lemma": "соба́ка",
  "pos": "NOUN",
  "labels": ["f", "anim"],
  "senses": [ { "glosses": ["dog"] } ]
}
```

Japanese entry with `reading`:

```jsonc
{
  "lemma": "母",
  "pos": "NOUN",
  "pron": { "ipa": ["[ha̠ha̠]"], "reading": "はは" },
  "senses": [ { "glosses": ["mother"] } ]
}
```

Compound with `segments` (`word` `Rettungsfonds`, `lang` `de_en`):

```jsonc
{
  "entries": [
    { "lemma": "die Rettung", "pos": "NOUN", "senses": [ { "glosses": ["rescue"] } ] },
    { "lemma": "der Fonds", "pos": "NOUN", "senses": [ { "glosses": ["fund"] } ] },
    { "lemma": "der Fond", "pos": "NOUN", "senses": [ { "glosses": ["back (of a vehicle)"] } ] }
  ],
  "segments": [
    { "text": "Rettung", "entries": [0] },
    { "text": "s" },
    { "text": "fonds", "entries": [1, 2] }
  ]
}
```

Stem and ending (`word` `talossa`, `lang` `fi_en`):

```jsonc
{
  "entries": [
    { "lemma": "talo", "pos": "NOUN", "senses": [ { "glosses": ["house"] } ] },
    { "lemma": "-ssa", "pos": "X", "senses": [ { "glosses": ["in"] } ] }
  ],
  "segments": [
    { "text": "talo", "entries": [0] },
    { "text": "ssa", "entries": [1] }
  ]
}
```

Prefixes before the stem (`word` `ninakupenda`, `lang` `sw_en`). The stem's entry is ranked first, so the client opens on `pend`:

```jsonc
{
  "entries": [
    { "lemma": "-penda", "pos": "VERB", "senses": [ { "glosses": ["to love"] } ] },
    { "lemma": "ni-", "pos": "X", "senses": [ { "glosses": ["I"] } ] },
    { "lemma": "-na-", "pos": "X", "senses": [ { "glosses": ["present tense"] } ] },
    { "lemma": "-ku-", "pos": "X", "senses": [ { "glosses": ["you (object)"] } ] }
  ],
  "segments": [
    { "text": "ni", "entries": [1] },
    { "text": "na", "entries": [2] },
    { "text": "ku", "entries": [3] },
    { "text": "pend", "entries": [0] },
    { "text": "a" }
  ]
}
```

Rules:

- `entries`: a server should send one per headword (`lemma`, `pos`, `labels` and `pron.reading` together); the client does not rely on it. Different grammar means separate entries (`der See` "lake", `die See` "sea"), and so does a different reading (`力` as ちから and as りき). Different meanings of one headword are senses of one entry (`die Mutter` "mother" and "nut"). The client keeps the server's order and does not merge entries.
- `lemma`: the headword as a learner's dictionary prints it, with marks texts leave out (`соба́ка`) and, where the server chooses, what is learned with the word (`der Hund`, `der/das Bonbon`). Grammar a dictionary prints as an abbreviation goes in `labels`.
- `pos`: a Universal Dependencies tag (`NOUN`, `VERB`, `ADJ`, ...), or `X` when none fits.
- `labels`: optional tags shown next to the headword, mostly grammar (`["f", "anim"]`, `["pf"]`), and the form that matched (`["3sg pres"]`). The server picks the strings, and the client shows them without interpreting them. A server may explain them (see Label explanations).
- `senses[].glosses`: at least one string. What the word means in this sense, in the pair's target language: one or more equivalents (`["mother"]`), or a definition, which a same-language pair always sends.
- `senses[].see`: optional words the sense points at (`"synonym of grządka"` → `["grządka"]`). It names a word, never a sense, written as its entry's `lemma` (`der Hund`, not `Hund`), one per lemma when there are several (`der See`, `die See`). Each must have an entry on this server. Following one is a lookup of that string that shows only entries with that `lemma`. Clients may ignore it.
- `sources`: optional links to the pages the entry is built from (the word's Wiktionary page; two when a meaning was copied from a base word's page). The client shows a link only when its origin is the origin of a `url` in the metadata `sources`, and names it with that source's `name` (see Sources). An entry the server wrote itself has none.
- `pron`: optional, and so is every field in it.
    - `ipa`: IPA strings.
    - `audio`: recordings, as links, never embedded data.
        - `url`: the file. Clients play it only when the user asks, over `https`, from the server's own host or `upload.wikimedia.org`.
        - `source`: optional link to the page that credits the recording (its file page on Wikimedia Commons). Shown under the same rule as an entry's `sources`. Two items with the same `source` are one recording in two formats.
    - `reading`: the word in a script the user can sound out (kana for Japanese, Latin letters for Cyrillic).
- `segments`: optional split of the word into written pieces (`Rettung` + `s` + `fonds`) or logical ones (`talo` + `ssa`). The entries mean the same with or without it.
    - `text`: joined in order, the texts equal the word exactly as sent, before NFC or case folding. One exception: in a script without spaces the client may send only a piece of the word (`食` of `食べました`). The server then names the whole word as the one-string `match` of its first entry, and the texts equal that string instead of `word` (see Context-match).
    - `entries`: optional indexes into `entries`, each entry in at most one segment. Entries in no segment belong to the whole word and come first. Within a segment, entries keep the top-level ranking. A segment without entries (the linking `s`) has no meaning of its own.
    - The main segment is the one that holds the first entry that is in a segment. The client shows it first, together with the whole-word entries; the other segments stay in written order and are shown when the user picks them. A server ranks first the entries of the piece that carries the word's meaning (the stem `pend` of `ninakupenda`, though `ni`, `na` and `ku` are written before it). A server with no such preference lists entries in segment order, which makes the first segment the main one.
    - The server sends one split, the one it finds most likely, and no alternatives. The split is a flat list: a piece has no pieces of its own, and how finely to cut is the server's choice (`Haupt` + `bahnhofsuhr`).
    - A response may hold a split and entries for the whole word together (`送る` for `お送り`, next to the pieces `お` and `送り`).
    - Endings are ordinary entries: a canonical `lemma` (`-ssa` for `ssa` and `ssä`), their meaning as the gloss, and `pos` `X` unless a UD tag fits (`AUX`). The segment `text` stays the surface form.
- Not in v1: a word's inflected forms and inflection tables, and usage labels (archaic, colloquial). The form that matched may be a label (see Note: tokenizing and deinflection).
- Not found: `200` with `{"entries": []}`. Errors: `400` with `{"error": "bad_request"}` or `{"error": "unsupported_lang"}`.

### Lookup context (optional)

A server that accepts `context` lists `"context"` in `features`. One that also returns `match` lists `"context-match"` as well. A server that does not list `context` ignores it and answers the plain lookup.

Example:

```jsonc
{ "word": "Tomaten", "lang": "de_en", "context": "Er hat Tomaten auf den Augen." }
```

- `context` is one sentence or subtitle line that contains `word`. When the sentence is longer than 500 characters, the client sends the part of it around `word`: the clause containing it, cut at clause punctuation (`,` `;` `:`, the full-width `，` `、` `；` `：`, the Arabic `،` `؛`) or a dash, together with whole neighbouring clauses while it fits; or else the words around `word`, or in text without spaces the characters around it. The server treats it like a sentence; a phrase is only found inside it.
- The client sends `context` only to servers that list it, and per server only as the user chose when binding: from all sites, from all sites except ones the user blocked (the default), or never.
- `at` (optional, only with `context`): the 1-based occurrence of `word` in `context` that was clicked. Both sides count exact copies of `word` in the `context` string as sent: left to right, without overlap, case-sensitive, before any normalization. Copies inside longer words count too (`кот` inside `который`, `восток` inside `северо-восток`), so client and server get the same number without agreeing on word boundaries. Example: `"word": "восток", "context": "...", "at": 2`. The server keeps only phrase entries matched at that occurrence. Without `at`, the server may be conservative with words that occur more than once. `at` applies with or without `context-match`.
- Multi-word entries (phrases, idioms) are ordinary entries with a multi-word `lemma`. Entries that match the context come first, phrases before single words.
- Servers must not log `context`. That includes any log of request bodies, at the server and at a proxy in front of it.

#### Context-match (`match` on entries)

When the server lists `context-match` and the request has `context`, each entry that is in the response because of the context (a phrase, a hyphen-joined word, a separable verb) carries `match`: the parts of `context` it covers, in order. The word's own entries have no `match`. There is no request parameter for it.

Example for `context` = `Тебе надо держать все время ухо востро, пока ты в этом городе.`:

```jsonc
{
  "entries": [
    {
      "lemma": "держа́ть у́хо востро́",
      "pos": "VERB",
      "senses": [ { "glosses": ["to keep an eye peeled"] } ],
      "match": ["держать", "ухо востро"]
    },
    { "lemma": "у́хо", "pos": "NOUN", "senses": [ { "glosses": ["ear"] } ] }
  ]
}
```

More examples:

- `word=fand`, `context="Die Sitzung fand gestern statt."` → stattfinden, `match` `["fand", "statt"]`
- `word=Kraft`, `context="Das Gesetz tritt morgen in Kraft."` → in Kraft treten, `match` `["tritt", "in Kraft"]`
- `word=miejsce`, `context="Miało to miejsce wczoraj."` → mieć miejsce, `match` `["Miało", "miejsce"]`
- `word=кого`, `context="Позови кого-нибудь."` → кто́-нибудь, `match` `["кого-нибудь"]`
- `word=食`, `context="昨日、寿司を食べました。"` → 食べる, `match` `["食べました"]`

Rules:

- `match` is a list of non-empty strings, each an exact substring of `context` as sent, before any normalization (NFC, case folding, stress marks), so the client finds it with a plain string search. No character offsets.
- Each string is a contiguous run of the entry's words, with the spaces or hyphens between them as written. Words of the sentence that are not part of the entry separate the strings (`["держать", "ухо востро"]`); adjacent words of the entry stay in one string. The strings appear in `context` in this order.
- One of the strings contains the clicked occurrence of `word`. When `word` occurs more than once in `context`, the client sends `at`, and the server returns only entries matched at that occurrence. `match` carries no offsets, so the client cannot tell by itself which occurrence the server matched; `match` is what it highlights.
- When a `match` string appears more than once in `context`, the client anchors on the clicked occurrence and places the strings in order so that one of them covers it.
- `match` is not for compounds inside one written word; that is what `segments` is for.
- In scripts without spaces the client's tokenizer may give only a piece of a word (`食` or `べ` of `食べました`, `し` of `勉強しています`). The server may then return the whole word as a one-string `match`, and `segments`, if sent, split that string.
- While the popover is open, the client underlines or tints the matched parts in the text.

Example: the user clicks `勉強` in `私は毎日日本語を勉強しています。` (`word` `勉強`, `lang` `ja_en`, with `context`). The tokenizer stops at `勉強`, but the word is `勉強しています`. The server sends the whole word as the `match` of its first entry, and `segments` splits it into its pieces:

```jsonc
{
  "entries": [
    { "lemma": "勉強する", "pos": "VERB", "pron": { "reading": "べんきょうする" }, "senses": [ { "glosses": ["to study"] } ], "match": ["勉強しています"] },
    { "lemma": "勉強", "pos": "NOUN", "pron": { "reading": "べんきょう" }, "senses": [ { "glosses": ["studying, study"] } ] },
    { "lemma": "する", "pos": "VERB", "senses": [ { "glosses": ["to do"] } ] },
    { "lemma": "ている", "pos": "AUX", "senses": [ { "glosses": ["-ing; ongoing action or state"] } ] },
    { "lemma": "ます", "pos": "AUX", "senses": [ { "glosses": ["polite"] } ] }
  ],
  "segments": [
    { "text": "勉強", "entries": [1] },
    { "text": "し", "entries": [2] },
    { "text": "てい", "entries": [3] },
    { "text": "ます", "entries": [4] }
  ]
}
```

### Note: tokenizing and deinflection

- The client only finds the piece of text the user clicked, with the browser's word segmenter. It needs no tokenizer per language, though one may be added for some languages later. The server receives that piece as it appears in the text, plus `context` as described above.
- What the piece belongs to in the sentence is found on the server, from `context`: German separable verbs, phrases, hyphen-joined words, and word boundaries in Japanese and Chinese.
- A segmenter without a dictionary cuts inflected words short in scripts without spaces (`Intl.Segmenter` splits `食べました` into `食`, `べ`, `ま`, `した`). With `context`, the server finds the whole word around the click and returns it as `match`; without `context` it answers the piece as sent.
- Deinflection happens on the server: an inflected word (`Mütter`) returns the entries of its base form (`Mutter`). The server may name the form that matched with `labels` on that entry, explained like any other label (`va` returns `aller` with `3sg pres` and `2sg imp`, since it is both; `並んで` returns `並ぶ` with `て form`). It names only the matched form, never the rest of the word's forms.
- The server returns all matching entries. Picking the right sense is out of scope for this protocol.
