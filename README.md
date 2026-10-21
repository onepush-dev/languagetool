# LanguageTool — self-hosted grammar, style and spell checking API (v6.8, 30+ languages, works with the browser add-ons). Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/TEMPLATE_CODE)

[LanguageTool](https://github.com/languagetool-org/languagetool) (LGPL-2.1) is the open-source proofreading engine behind the LanguageTool browser extensions, LibreOffice and many editor plugins. It finds grammar, style, punctuation and spelling mistakes in more than 30 languages (English, German, French, Spanish, Dutch, Portuguese, Polish, Russian, Ukrainian, Catalan and more) and goes well beyond a spell checker. This template runs its HTTP API server (`/v2/check`, `/v2/languages`) at **6.8** (latest stable, released 2026-05-05) as a single Railway service. It needs **no deploy-time input** and no volume: deploy, then point your browser add-on or editor at `https://<your-domain>/v2`.

## Why this template
- **Current, pinned release.** LanguageTool stopped publishing release ZIPs after 6.6 and ships no official Docker image, so this uses [`erikvl87/languagetool`](https://github.com/Erikvl87/docker-languagetool) (6M+ pulls, built from the upstream `v6.8` tag), pinned by tag **and** digest. A redeploy never jumps to another version behind your back. (The other popular image, `meyay/languagetool`, deletes older tags/digests whenever it rebuilds, so a pinned template on it would break.)
- **Works with the browser extensions out of the box.** HTTPS comes from Railway, CORS is open (`Access-Control-Allow-Origin: *`), and the API listens in public mode, so the LanguageTool add-ons for Chrome/Firefox/Edge/Safari, LibreOffice, Obsidian, LTeX (VS Code), Thunderbird and others can use it as their server.
- **Automatic language detection.** The fastText language-identification model is baked into the image, so `language=auto` works with no download at boot.
- **Sensible limits for a public URL.** Texts up to 100,000 characters, 60 s per check, and a per-client request limit (see notes). JVM heap 256 MB to 1 GB.
- **Optional n-gram data, off by default.** Set `NGRAM_LANGUAGES=en` (or `de`, `es`, `fr`, `nl`) and add a volume at `/ngrams`: the data is downloaded in the background while the API keeps serving, then the service restarts once to load it. LanguageTool's `languageModel` is only switched on when data is actually there.
- **Not running as root.** The start command runs as root only to own an optional `/ngrams` volume, then drops to the image's unprivileged `languagetool` user.
- **Stateless.** Rules, dictionaries and models ship in the image; nothing needs to persist, so there is no volume to pay for unless you opt into n-grams.

## Service
| Service | Source | Port | Volume | Notes |
|---|---|---|---|---|
| LanguageTool | `erikvl87/languagetool:6.8@sha256:ef8fa12cbd485166c9ceeb7139d76d56d07707a624da6bb1fc1fbb5411750527` | `8010` (public) | none (optional `/ngrams`) | Health `/v2/languages` (timeout 300 s), restart `ALWAYS`, start command below |

### Start command
The image has no ENTRYPOINT, so the command ends by running upstream's own `/LanguageTool/start.sh` (which writes every `langtool_*` variable into `config.properties` and starts the server on port 8010). A readable version is in `railway/start.sh`, the one-line version in `railway/start-command.txt`.
```
/bin/bash -c 'cd /LanguageTool; N=/ngrams; U=https://languagetool.org/download/ngram-data; ROOT=0; [ "$(id -u)" = 0 ] && ROOT=1; [ $ROOT = 1 ] && [ -d $N ] && chown languagetool:languagetool $N; JOBS=""; for l in $(echo "${NGRAM_LANGUAGES:-none}" | tr "A-Z," "a-z "); do case $l in none|off|false) continue ;; en) f=ngrams-en-20150817.zip ;; de) f=ngrams-de-20150819.zip ;; es) f=ngrams-es-20150915.zip ;; fr) f=ngrams-fr-20150913.zip ;; nl) f=ngrams-nl-20181229.zip ;; *) echo "ngrams: unknown language $l (choose from en,de,es,fr,nl)" >&2; continue ;; esac; if [ -d $N/$l ]; then echo "ngrams: $l already on the volume"; elif [ -d $N ] && [ -w $N ]; then JOBS="$JOBS $l:$f"; else echo "ngrams: $l requested but $N is not a writable volume; add a volume at $N (see README)" >&2; fi; done; if [ -n "$JOBS" ]; then P=$$; ( NEW=0; for j in $JOBS; do l=${j%%:*}; f=${j#*:}; echo "ngrams: downloading $f in the background (LanguageTool keeps serving meanwhile)"; rm -rf $N/.$l.zip $N/.$l.tmp; if curl -fsSL --retry 5 -o $N/.$l.zip $U/$f && unzip -q -o $N/.$l.zip -d $N/.$l.tmp && [ -d $N/.$l.tmp/$l ]; then mv $N/.$l.tmp/$l $N/$l; rm -rf $N/.$l.zip $N/.$l.tmp; [ $ROOT = 1 ] && chown -R languagetool:languagetool $N/$l; echo "ngrams: $l installed"; NEW=1; else echo "ngrams: $l download or unzip failed (check free space on the volume)" >&2; rm -rf $N/.$l.zip $N/.$l.tmp; fi; done; if [ $NEW = 1 ]; then echo "ngrams: restarting so LanguageTool loads the new data"; kill $P; fi; ) & fi; if [ -z "$langtool_languageModel" ]; then for l in en de es fr nl; do [ -d $N/$l ] && export langtool_languageModel=$N && break; done; fi; [ -n "$langtool_languageModel" ] && echo "ngrams: languageModel=$langtool_languageModel"; if [ $ROOT = 1 ]; then exec su languagetool -s /bin/sh -c "echo \"running LanguageTool as \$(id -un)\"; exec bash start.sh"; fi; exec bash start.sh'
```

## Variables
Every value is also in `env.example` (and `railway/template-variables.env`). There are no secrets.

| Variable | Template value | Notes |
|---|---|---|
| PORT | `8010` | port the public domain routes to (fixed by the image) |
| RAILWAY_RUN_UID | `0` | start as root only to own an optional `/ngrams` volume; LanguageTool itself runs as `languagetool` |
| Java_Xms | `256m` | initial Java heap |
| Java_Xmx | `1g` | maximum Java heap; raise to `2g`+ for many languages or heavy traffic |
| NGRAM_LANGUAGES | `none` | n-gram data to download (`en,de,es,fr,nl`); needs a volume at `/ngrams` |
| langtool_trustXForwardForHeader | `true` | take the client address from Railway's `X-Forwarded-For` for request limits |
| langtool_requestLimit | `120` | checks per client per period (0 = unlimited) |
| langtool_requestLimitPeriodInSeconds | `60` | request-limit window |
| langtool_maxTextLength | `100000` | longest text per check, in characters (longer texts get HTTP 413) |
| langtool_maxCheckTimeMillis | `60000` | stop a single check after 60 s |
| langtool_cacheSize | `1000` | sentence results kept in memory for faster rechecks |
| langtool_pipelineCaching | `true` | reuse prepared checking pipelines between requests |

Any other [HTTPServerConfig](https://languagetool.org/development/api/org/languagetool/server/HTTPServerConfig.html) option can be set the same way: `langtool_` + the property name (e.g. `langtool_pipelinePrewarming=true`, `langtool_disabledRuleIds=WHITESPACE_RULE`, `langtool_maxWorkQueueSize=50`). `langtool_fasttextModel` / `langtool_fasttextBinary` are managed by the image and must not be set.

## How to use
1. Click **Deploy on Railway** and wait for the service to turn healthy (about 30 seconds).
2. Test it: `curl -d language=en-US --data-urlencode "text=This are a test." https://<your-domain>/v2/check` returns a JSON list of matches.
3. **Browser extension:** open the LanguageTool add-on settings → *Advanced settings* → *LanguageTool server* → *Other server* and enter `https://<your-domain>/v2`.
4. **LibreOffice:** *Tools → Options → Language Settings → LanguageTool Server* → enable and enter `https://<your-domain>/v2`. **LTeX (VS Code):** set `ltex.languageToolHttpServerUri` to `https://<your-domain>/`. **Obsidian LanguageTool plugin:** custom server `https://<your-domain>`.

## Optional: n-gram data
N-grams improve detection of commonly confused words (their/there, effect/affect). They are large: English is an 8.9 GB download that unpacks to 15.4 GB (about 25 GB free space needed during install); German 1.7 → 3.3 GB, Spanish 1.7 GB, French 1.8 GB, Dutch 1.2 → 2.5 GB.
1. Add a volume to the service, mount path `/ngrams`, sized for the languages you want (e.g. 30 GB for English; needs a plan that allows it).
2. Set `NGRAM_LANGUAGES` to e.g. `en` and redeploy. The logs show `ngrams: downloading…`, then `ngrams: en installed`, and the service restarts once to load it (`ngrams: languageModel=/ngrams`).
3. Give the JVM a bit more room if needed (`Java_Xmx=2g`).

**Upstream caveat (6.8):** once `languageModel` is set, LanguageTool skips the spelling rules for every language that has **no** n-gram data on the volume (it logs `WARN: ngram index dir /ngrams/xx not found`). Grammar rules keep working. Only enable n-grams if you mainly check the languages you downloaded. To go back, set `NGRAM_LANGUAGES=none` and delete the volume.

## Notes
- **Who can use it.** LanguageTool's open-source server has no API keys or login: anyone who knows your URL can send text to it (that is also what lets the browser add-ons use it). Text is processed in memory and not stored; the server logs request metadata (sizes, language, timing) but not text. If you need it private, remove the public domain and call it from other Railway services over `http://<service>.railway.internal:8010`, or put an authenticating proxy in front.
- **Request limit is best effort.** LanguageTool keys the limit on the whole `X-Forwarded-For` value, which on Railway is `<client IP>, <edge IP>` and the edge hop varies, so the effective per-client limit is looser than 120/min. `langtool_maxTextLength` and `langtool_maxCheckTimeMillis` are always enforced.
- **Memory.** Idle usage is a few hundred MB; each language loads its rules and dictionaries on first use. Raise `Java_Xmx` if you see `OutOfMemoryError` in the logs.
- **No Premium features.** Premium-only rules and the AI rewriting of languagetool.org are not part of the open-source server. The responses include a harmless `premiumHint` field.
- **Custom domain.** Add it in Settings → Networking; no variable changes needed.
- **Upgrade.** Change the image tag and digest to a newer `erikvl87/languagetool` release.
- **Upstream docs:** [HTTP server](https://dev.languagetool.org/http-server), [HTTP API](https://languagetool.org/http-api/), [n-gram data](https://dev.languagetool.org/finding-errors-using-n-gram-data), [image README](https://github.com/Erikvl87/docker-languagetool).

## License
LanguageTool is LGPL-2.1 licensed by the LanguageTool team and contributors; the Docker image is by Erik van Leeuwen. This repository only holds Railway configuration.
