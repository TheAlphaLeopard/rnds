# Where the game files come from

`index.html` is the whole front-end of this Unity WebGL build. It fetches every
other asset at runtime; there are no hard-coded absolute URLs in the loader or
framework (only browser bug-report links). Paths are relative to the URL that
serves `index.html` (`const buildUrl = 'Build';`).

## Current setup

| Asset | Source |
|---|---|
| `index.html`, `Build/WebBuild.loader.js`, `main.js` | this repo |
| `Build/WebBuild.data.part-aa` … `.part-af` (6 files) | this repo (downloaded from the off-site host) |
| `Build/WebBuild.wasm.part-aa` … `.part-ae` (5 files) | this repo (downloaded from the off-site host) |
| Framework (`frameworkUrl`) | `https://truffled.lol/f/WxUUCwMfShYbBxsCFUMMChAXDUgOGAgI/Build/WebBuild.framework.js`, with a fallback to `main.js` (see below) |
| `StreamingAssets/` | relative to the page URL, not to `Build/` |

If the page were served at `https://google.com/index.html`, the local files
resolve to `https://google.com/Build/<name>`; if it were at
`https://google.com/rnds/index.html`, they resolve to
`https://google.com/rnds/Build/<name>`.

## The 11 split parts

`index.html:62` fetches `${buildUrl}/${baseName}.part-${suffix}`; the suffix lists
are in `index.html:84-85`:

- data: `WebBuild.data.part-aa`, `-ab`, `-ac`, `-ad`, `-ae`, `-af` (117,422,091 bytes total)
- wasm: `WebBuild.wasm.part-aa`, `-ab`, `-ac`, `-ad`, `-ae` (104,558,191 bytes total)

They are concatenated in exactly that order into Blob URLs, so
`WebBuild.data` / `WebBuild.wasm` are never requested as single files. The merged
data starts with `UnityWebData1.0\0` and its header offsets add up exactly to the
total size; the merged wasm starts with `\0asm`.

## Framework URL and CORS

`frameworkUrl` is set to the off-site URL above. `Build/WebBuild.loader.js:375`
fetches it with `fetch()` and injects it as a Blob script; the file must define the
global `unityFramework` (it does).

The off-site host responds with `content-type: application/javascript` and
`content-length: 511241`, but **no `Access-Control-Allow-Origin` header**. That means
a browser will block the fetch when the page is served from any other origin
(`cross-origin-resource-policy: cross-origin` does not replace CORS). To keep the
game working, `index.html` probes that URL and falls back to the framework bundled
in this repo (`main.js`, which is the same file beautified) when the fetch is
blocked. If you want the remote copy to be used from another origin, the host must
add `Access-Control-Allow-Origin: *` (or your page's origin), or the page must be
served from `truffled.lol` itself.

## If you want every Build file from that host instead

Set `const buildUrl = 'https://truffled.lol/f/WxUUCwMfShYbBxsCFUMMChAXDUgOGAgI/Build';`
in `index.html:54` and set `streamingAssetsUrl` to an absolute URL too. Note that
all 12 `fetch()`-ed URLs (11 parts + framework) would then need CORS enabled on
that host; `Build/WebBuild.loader.js` is a plain `<script>` tag and needs no CORS.
Serve the parts as `application/octet-stream` and the framework as JavaScript with
no bogus `Content-Encoding: br`/`gzip`.
