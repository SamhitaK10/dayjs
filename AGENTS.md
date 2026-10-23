# AGENTS.md: Day.js

Day.js is a ~2 KB immutable date library that copies Moment.js's API. The core is deliberately tiny. Most features live in opt-in plugins and locales. Two constraints matter more than anything else: the minified bundle must stay under the size budget, and behavior must match Moment.

## Layout
- `src/index.js`: the whole core. It holds the `Dayjs` class, the `dayjs()` factory, and global locale state. `src/utils.js` is exposed to plugins as `this.$utils()` under one-letter keys (`s` pad, `p` prettyUnit, `u` isUndefined, …). `src/constant.js` holds unit names and regexes.
- `src/plugin/<camelName>/index.js`: one directory per plugin, always `index.js`.
- `src/locale/<key>.js`: one flat file per locale.
- `types/`: hand-written `.d.ts` files. `types/plugin/<camelName>.d.ts` mirrors each plugin.
- `test/`: Jest. Its layout mirrors `src/` (`test/plugin/`, `test/locale/`), plus `test/issues/` for regressions.
- `docs/` holds only stub pages pointing to day.js.org. The real docs live in a separate repo, so don't document features here.

## Commands (Node LTS, npm, run from repo root)
- Install: `npm ci`
- Lint: `npm run lint` (ESLint 4 + airbnb-base; also runs as a pre-commit hook)
- One test file: `npx jest test/plugin/weekday.test.js`. This is the local "done" check.
- One file in a specific zone (works on every OS): `npx cross-env TZ=Europe/Paris npx jest test/plugin/timezone.test.js`. Don't use inline `TZ=… npx jest`; on Windows Git Bash it never reaches Node and silently tests local time.
- Full suite, as CI runs it: `npm test`. This re-runs the timezone tests under Pacific/Auckland, Europe/London, America/Whitehorse, Europe/Paris and America/New_York, then runs everything with a **100% line-coverage gate**. On Windows it stops at cmd's interactive `date` prompt. Outside CI's zones it can fail for reasons unrelated to your change; e.g. 4 timezone-plugin DST tests fail in America/Los_Angeles.
- Build: `npx cross-env NODE_OPTIONS=--openssl-legacy-provider npm run build`. Without the flag, the `size` step (size-limit 0.18 → webpack 4) fails on modern Node.
- ESM build: `npm run babel` (run after `build`)

## Conventions
- Plugin shape: `export default (o, c, d) => { c.prototype.foo = function () { … } }`, where `o` = options, `c` = the Dayjs class, `d` = the `dayjs` factory. Plugins don't import `dayjs`. Use `function`, not an arrow, for prototype methods, because they need `this`.
- Instances are immutable. Return a new instance via `this.clone()`, `this.set()`/`add()`, or `this.$utils().w(...)`. Never mutate `this.$d` from a public method. Internal fields are `$`-prefixed (`$d`, `$y`, `$M`, `$L`, `$u`, `$x`). Read them, don't add new ones casually.
- Locale files: line 1 must be `// <English Name> [<key>]` (the build and `test/locale/keys.test.js` parse it by regex). `name` must equal the lowercase file name. The file ends with `dayjs.locale(locale, null, true)` then `export default locale`. Locales import `'dayjs'`, which `test/__mocks__/dayjs.js` maps to `src/` in tests.
- Tests import from `../../src`, never from a built file. Use `moment` as the expected-value oracle (`expect(dayjs(x).foo()).toBe(moment(x).foo())`). Wrap time-dependent tests in `MockDate.set` in `beforeEach` and `MockDate.reset()` in `afterEach`. Reset any global locale you change.
- Style: no semicolons, no trailing commas, 2-space indent, extensionless imports.
- Commits use semantic messages (`fix:`, `feat:`). They drive semantic-release versioning.

## Guardrails
- Never edit build output: `/locale`, `/plugin`, `/esm`, `/dayjs.min.js`, `/index.d.ts`, `locale.json` at the repo root. All of these are generated and gitignored. Root `locale/` ≠ `src/locale/`. Root `index.d.ts` is copied from `types/index.d.ts`.
- A new plugin needs `types/plugin/<name>.d.ts`, or `test/plugin/declarations.test.js` fails.
- Don't run `npm test -- <file>` to test one file. It still runs all the timezone passes, then applies the 100%-coverage gate to that one file, and fails. Use `npx jest <file>`.
- Every new line in `src/` needs a test covering it, or `npm test` fails the coverage gate. Adding code to core costs bytes. Prefer a plugin unless the change belongs in core.
- Don't edit `CHANGELOG.md` or the `version` in `package.json` (`0.0.0-development`). semantic-release owns both.
- Don't modify `test/browser.spec.js` (Sauce Labs smoke test; its header says not to).
- `dev` is the default branch and the PR target. A push to `master` triggers an npm release.
