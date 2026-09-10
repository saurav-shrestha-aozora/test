# Questions for the maintainer — `draw-tool`

Context: I'm new to this repo and hitting friction in three areas — **installing**, **running it locally**,
and **one feature I want to add**. Grouped below so you can answer whichever parts are quickest.

My environment, for reference:

| | |
|---|---|
| OS | Windows 11 (10.0.26200) |
| Shell | PowerShell / Git Bash |
| Node in this repo | v10.24.1 (npm 6.14.12) |
| Node elsewhere on my machine | v24.18.0 (npm 11.16.0) |
| Repo state | `master` @ `9560094`, `node_modules/` present, `build/` not committed |

---

## 1. Installing dependencies

The dependency set in [package.json](package.json) is pinned to an old toolchain (webpack 1.15, babel 6,
fabric 1.x), and installing it on a modern machine is where I lose the most time.

1. **What Node and npm version do you build this with?** I currently have to keep Node 10 around just for
   this repo. Is Node 10 still the supported version, or is there a newer one that works? On Node 17+ the
   webpack 1 build fails with the OpenSSL 3 `digital envelope routines::unsupported` error — do you work
   around that with `NODE_OPTIONS=--openssl-legacy-provider`, or is the answer simply "stay on old Node"?
2. **npm or yarn?** The repo has *both* [package-lock.json](package-lock.json) (lockfileVersion 1) and
   [yarn.lock](yarn.lock). Which one is authoritative? Should I commit lockfile changes when I install, or
   leave them alone?
3. **Is `npm ci` supposed to work?** Or is a plain `npm install` (possibly with `--legacy-peer-deps`)
   the expected path?
4. **`.npmrc`** — is there a required registry / auth / flag config that isn't committed? I ended up creating
   a local `.npmrc` to get the install through. If there's a canonical one, could it be committed (or the
   required settings documented)?
5. **`fabric` is installed from a personal fork:**
   ```json
   "fabric": "github:thiennx-nws/fabric.js#1.x"
   ```
   - Why the fork rather than upstream `fabric@1.x`? What patches does it carry?
   - Is that repo guaranteed to stay public? If it's ever deleted or made private, every fresh install breaks.
     Would you consider vendoring it or publishing it to a private registry?
   - It needs `git` on PATH to install — is `git+https` the intended protocol, or does it need SSH keys?
6. **`node-gyp@^3.8.0` is a direct devDependency.** What actually needs it? node-gyp 3.x wants Python 2.7 and
   MSVC build tools, which is a large Windows-only setup cost. If nothing in the build compiles native code
   any more, can it be dropped?
7. Are there **prerequisites outside npm** I should install first (Python, Visual Studio Build Tools,
   `windows-build-tools`, a specific Git version)? A one-line "prereqs" section in the README would help a lot.

> **Note:** I had saved the exact install error output in a scratch file, but it's no longer in my working
> tree, so the questions above are based on the dependency set rather than the specific stack trace. I'll
> re-run a clean `rm -rf node_modules && npm install` and paste the real output if that's more useful.

---

## 2. Running the project locally

There's no `start` or `dev` script, so I've been guessing at the loop.

1. **What is the actual dev loop?** My current guess is:
   ```
   npm run watch          # webpack --watch -> build/drawtool.js
   # then serve examples/ over http and open index.html
   ```
   Is that right, or is there a step I'm missing?
2. **The npm scripts assume a POSIX shell.** These fail in PowerShell/cmd:
   ```json
   "watch":  "NODE_ENV=development webpack --watch",
   "build":  "npm --no-git-tag-version version patch && NODE_ENV=production webpack; NODE_ENV=development webpack",
   "copy2upt": "cp build/drawtool.js ../up-t-web-draw-1/src/draw-tool/"
   ```
   - Do all the Windows devs just run these from Git Bash / WSL?
   - Would you accept a PR adding `cross-env` (and a Node-based copy) so the scripts work cross-platform?
3. **How do you serve `examples/`?** [examples/index.html](examples/index.html#L111) loads
   `../build/drawtool.js`, and the sample JSON in [examples/index.js](examples/index.js) references
   `http://127.0.0.1:8080/img/...`, so `file://` clearly won't work. Is the expected server
   `webpack-dev-server` (it's a devDependency but no script uses it), `http-server`, or something internal?
   What port and document root?
4. **`build/` is in [.gitignore](.gitignore) but `examples/` depends on it.** So a fresh clone can't open the
   example until it builds successfully. Is that intended? Should the README say "run `npm run watch` first"?
5. **`npm run build` bumps the version on every run** (`npm --no-git-tag-version version patch`). Is that meant
   to run only on release, or on every local build? Right now a routine local build dirties
   [package.json](package.json), which makes diffs noisy.
6. **How does the built file reach the consuming apps?** The `copy2upt` / `copy2orilab` / `copy2budgets` /
   `copy2nail` scripts copy `build/drawtool.js` into sibling repos by relative path — which assumes a specific
   folder layout on disk. What's the expected directory structure? Is publishing to a registry an option
   instead, or is the copy step deliberate?
7. **Is there a test suite or any smoke check?** There's no `test` script. How do you verify a change to
   e.g. [Side.js](src/drawTool/Side.js) or [clip.js](src/drawTool/utils/clip.js) hasn't broken a product type,
   given how heavily the code branches on product (`is_nail`, `checkProductLaser`, penlight, acrylic, carpet…)?
8. **Docs:** `npm run docs` outputs to `build/docs/` via jsdoc. Is
   http://wiki.drawtool.s3-ap-northeast-1.amazonaws.com/index.html still the current published reference, and
   who regenerates it?
9. **Which product/side JSON should I use as a starting point?** The examples file has several `var json = ...`
   blocks with all but one commented out. Is there a canonical fixture set somewhere? (`data/` is empty in my
   checkout.)

---

## 3. Live-editing / real-time update feature

> **The question:** How can I add a live-editing feature to this project where users can edit the
> response/input and see the updated result in real time? The changes should appear immediately while typing,
> with debouncing to prevent unnecessary API calls, and streaming to display the updated response as it is
> generated. Are there any existing methods for this, or would it need new plumbing?

Concretely: a user types in an input (text content, or item options like size / color / position) and the
canvas — plus any server round-trip that depends on it — updates as they type, rather than on blur or on an
explicit "apply" click.

Specific questions:

1. **Is there already a supported "update as you type" path?** I can see `Items.updateItem(data)` in
   [Items.js:55](src/drawTool/Items.js#L55) and the `editing:entered` / `editing:exited` events. Is
   `updateItem` cheap enough to call on every keystroke, or is it intended for commit-time only?
2. **Which events should I subscribe to for live changes?** From `DrawTool.on(...)`
   ([DrawTool.js:715](src/drawTool/DrawTool.js#L715)) I can see `object:modified`, `option:update`,
   `item:option`, `draw:update`, `editing:entered`, `editing:exited`, `after:render`. Which of these fire
   *during* an edit vs. only at the end? Is there a per-character text-change event, or do I need to hook
   fabric's `text:changed` directly?
3. **Debouncing** — there's a [throttle.js](src/drawTool/utils/throttle.js) util in the repo, but nothing
   imports it. Was it built for exactly this and then abandoned? Is there a preferred delay for canvas
   re-render vs. for network calls, and should the debounce live inside the library or in the host app?
4. **Undo/redo interaction.** [DrawHistory.js](src/drawTool/DrawHistory.js) records changes — if I push an
   update on every keystroke, do I get one history entry per character? Is there a way to batch or suspend
   history during a live edit and commit a single entry when editing ends?
5. **Render cost.** `updateItem` and `insertPlainItem` both call `FabricCanvas.renderAll()`, and
   `insertPlainItem` temporarily flips `renderOnAddRemove`. For high-frequency updates, is `renderAll()` on
   every change acceptable, or should I batch? Any known perf cliffs at large canvas sizes (the examples use
   1800×1800)?
6. **Streaming / incremental results.** Where server-generated output is involved, is there any existing
   support for applying a *partial* result to the canvas (append/patch an item as chunks arrive), or is the
   library strictly "replace the whole item / re-`importJSON`"? `importJSON`
   ([DrawTool.js:1092](src/drawTool/DrawTool.js#L1092)) looks like a full-state replace — is there a
   lighter-weight partial-update entry point?
7. **Preview generation cost.** `Side.toSVG()` / `toDataURL()` show up in several paths — if the live preview
   needs one of those per update, is that the bottleneck? Any cheaper preview path you'd recommend?
8. **Would you take a PR for this?** If yes, where would you want the debounce/live-update layer to live —
   inside `DrawTool` as an opt-in mode, or entirely in the consuming app with the library staying as-is?

---

## 4. General / contribution

1. Is there a **branching + PR convention**? Recent history shows `feature/UPT-xxxxx-...` branches — is the
   ticket ID required?
2. Is there a **code style / lint config**? I don't see one committed, and formatting varies between files.
3. `DrawTool.js` (5,062 lines) and `Side.js` (4,484 lines) are large and product-branch heavy. Is there an
   intended refactor direction, or should new work follow the existing pattern?
4. Which **downstream apps** consume this library today, and which of them do I need to smoke-test against
   before merging a change?
