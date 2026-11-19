# Building a server

What a server has to do to speak the d0 lookup protocol, step by step. Each step links to the part of [PROTOCOL.md](PROTOCOL.md) with the exact rules; when this guide and the protocol disagree, the protocol is right.

A working server is [dict-server-ts](https://github.com/d0-lang/dict-server-ts), if you want to see one.

## The endpoints

| Endpoint | Needed | Answers with |
| --- | --- | --- |
| `GET /v1/get-key` | always | Your public key. Not signed. |
| `GET /v1/get-meta` | always | Your metadata, signed, with the client's nonce. |
| `POST /v1/lookup` | always | Entries for a word. |
| `GET /v1/get-labels?lang=…` | if you list `labels` in `features` | Explanations of your grammar labels. |

Bodies are JSON. Ignore request fields and query parameters you don't know: newer clients may send them.

## 1. A key you keep

Make one Ed25519 key pair for the server and keep it for as long as the server runs. Clients store your public key when the user adds your server. If the key changes, every client shows its users a warning that your server's identity changed, and pauses until they accept it.

So: back the private key up, keep it out of git, and use the same key on every machine that serves the same address.

→ [Server key](PROTOCOL.md#server-key), [Identity change](PROTOCOL.md#identity-change)

## 2. Signed metadata

`GET /v1/get-meta` comes with an `X-Nonce` header. Put that string in the `nonce` field, then sign the **exact bytes** of the body you send and put the base64 signature in `X-Signature`. Don't build the JSON twice: a client checks the signature against the bytes it received.

What goes in:

- `name`: how clients show your server.
- `languages`: the pairs you serve, `<word language>_<meaning language>` (`de_en`).
- `features`: what you support beyond plain lookups: `context`, `context-match`, `labels`.
- `verification_urls`: where you post news about the server, somewhere other than the server itself. Clients show these if your key ever changes.
- `sources`: whose data your answers come from, with licences, so clients can credit them. Needed for Wiktionary (CC BY-SA).
- `report_url`, optional: where users report a wrong or missing answer.
- `legal`: your privacy policy and terms (step 4).

→ [Metadata](PROTOCOL.md#metadata), [Sources](PROTOCOL.md#sources), [Reports](PROTOCOL.md#reports-optional)

## 3. Lookups

`POST /v1/lookup` with `word` and `lang`, and from clients that may send it, `context` and `at`.

- **Limits.** `word` is at most 64 characters and `context` at most 500, counted in UTF-16 code units. You may answer longer ones with `400` and `{"error": "bad_request"}`.
- **Deinflect.** An inflected word (`Mütter`) returns the entries of its base form (`Mutter`). Clients don't do this for you.
- **One entry per headword.** Headword, part of speech and labels together; different meanings are senses of one entry. `pos` is a Universal Dependencies tag, or `X`.
- **Nothing found** is `200` with `{"entries": []}`. A pair you don't serve is `400` with `{"error": "unsupported_lang"}`.
- **Compounds and endings**, if you split words: `segments`, whose texts join back into the word exactly as sent.
- **Pronunciation**: audio as links only, from your own host or `upload.wikimedia.org`. Clients play nothing else.
- **Links** in an answer (`sources` on entries, `source` on audio) are shown only when they point to a site you listed in your metadata `sources`.

→ [Word lookups](PROTOCOL.md#word-lookups)

## 4. Legal texts

A public server publishes a privacy policy and terms, links them in `legal`, and sends the sha256 (lowercase hex) of each one's source text. Clients store the hashes. **Any change to either text changes its hash, and every client then stops sending lookups until its user has seen the change**, so settle the texts before you go public. A change can carry a short summary of up to 500 characters (`privacy_change`, `tos_change`).

Without `legal`, clients tell users your server is for testing only, and send it no sentences unless the user agrees in that same message.

dict-server-ts has templates for both texts, written for EU law.

→ [Legal](PROTOCOL.md#legal)

## 5. Sentences (optional)

List `context` in `features`, and clients that the user allows will send the sentence around the word: the whole sentence, or the part around the word when it's longer than 500 characters. Use it to find phrases, separable verbs and word boundaries; put phrase entries first. `at` says which copy of the word was clicked when it appears more than once.

**Never log `context`**, at the server or at a proxy in front of it. It comes from whatever the user is reading.

With `context-match` in `features` too, each entry that's there because of the sentence carries `match`: the exact substrings of `context` it covers, in order, which clients underline.

→ [Lookup context](PROTOCOL.md#lookup-context-optional), [Context-match](PROTOCOL.md#context-match-match-on-entries)

## 6. Label explanations (optional)

If your entries carry grammar `labels` (`pf`, `anim`), list `labels` in `features` and explain them at `GET /v1/get-labels`: a name of up to 60 characters, and optionally a longer text, written in the pair's meaning language.

→ [Label explanations](PROTOCOL.md#label-explanations-optional)

## 7. Moving to a new address

If the server moves, `GET /v1/get-meta` at the old address answers with a signed `{"nonce": …, "new_source": "https://…"}`. Clients follow it with the same key and no warning.

→ [Domain move](PROTOCOL.md#domain-move)

## Before you go public

- [ ] The key is backed up, and every machine of the server uses it.
- [ ] `get-meta` is signed over the exact bytes sent, and echoes the nonce.
- [ ] `verification_urls` points somewhere other than the server.
- [ ] `sources` credits your data, with licences.
- [ ] The privacy policy and terms are published and settled; `legal` carries their hashes.
- [ ] Nothing logs request bodies or `context`, neither the server nor a proxy.
- [ ] Oversized requests get `400` before any lookup work.
