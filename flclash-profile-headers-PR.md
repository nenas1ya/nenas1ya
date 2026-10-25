# feat(profiles): send custom request headers with subscription updates

Target: chen08209/FlClash `main` (based on `v0.9.2`, 7fd417f).
Apply with `git am flclash-profile-headers.patch`.

## What
Per-profile custom HTTP headers that are sent when a URL profile is downloaded or updated. You edit them under **Edit profile → Request headers** (URL profiles only). They work the same way everywhere a subscription is fetched: add-from-link, manual and automatic updates, and the refresh before setup. All of these go through `Profile.update`.

## Why
Some subscription panels only answer clients that identify themselves. For example, the Remnawave device limit expects `x-hwid` (and optionally `x-device-os`, `x-ver-os`, `x-device-model`), and without it the panel returns an empty subscription. The feature is general and opt-in, and nothing is sent by default. Nothing in the code is specific to HWID.

## Notes
- **DB schema 21 → 22.** This adds a nullable `profiles.headers` column (a JSON string map, `NULL` when empty) through `_addColumnIfMissing`, so the step can run again safely. Schema 21 shipped in v0.9.2, so this is a new step.
- **Basic auth from the URL still wins.** Credentials in the URL's userinfo replace an `Authorization` header from the profile, matched case-insensitively.
- **`User-Agent`.** A `User-Agent` among the headers replaces the client's one (`globalState.ua`) for that profile only, because Dio's `headers.set` runs after `HttpClient.userAgent`.
- **Validation.** dart:io throws on header names that are not RFC 9110 tokens and on values outside printable ASCII or HTAB. `MapEditView` now takes optional `keyValidator` / `valueValidator` (single entry and batch mode) and `ignoreKeyCase`. Header names are compared case-insensitively. This adds one new string, `invalidCharactersTip`, in all 7 locales; the title reuses `requestHeaders`.
- **Body-framing headers are dropped.** `Content-Length` and `Transfer-Encoding` would break a bodiless GET in dart:io, so they are never sent.
- **Redirects.** Like curl and dart:io by default, custom headers follow a redirect to another host; only `Authorization` and `Cookie` are stripped there. Keep secrets in headers only for panels you trust.
- **Add flow is unchanged.** If a panel answers the first download with an empty body, the profile is still added, because an empty config is valid, and you can then add headers in the edit page, which downloads it again. A panel that returns an error status without the header still needs headers in the add form; that is left for a follow-up.
- **Changing headers in the edit page downloads the subscription again**, the same as changing the URL or the age key.

## Tests
- `test/models/profile_save_test.dart`: a real loopback download checks that the profile headers reach the server, that `User-Agent` is overridden, and that URL credentials beat a profile `Authorization`.
- `test/common/request_test.dart`: tests for `downloadTarget` merging, the dropped framing headers and the header validators.
- `test/database/migration_v1_to_v2_test.dart`: the v21 → v22 upgrade adds `headers` and existing rows read as `{}`. Every existing step now expects 22.
- `generated_contract_test`, `dao_coverage_test`: round trip of the new column.
- `profile_test`: old Profile JSON without `headers` still parses.
- `input_entries_test`: batch parsing reports invalid keys and values.
- `edit_profile_focus_test`: the headers row appears only for URL profiles.
