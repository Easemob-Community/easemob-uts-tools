# 1.0.1

- iOS: fix picker staging files being reclaimed by the system before `prepareFile` runs. Files are now copied into the app sandbox inside the picker callback (while the security-scoped access is still valid) and `prepareFile` returns the registered copy.
- iOS: `success`/`complete` callbacks are now deferred to the next main-runloop tick, so calling other plugin APIs synchronously inside callbacks works reliably (the bridge silently swallows native re-entrant calls made inside a native-to-JS callback).
- iOS: replace `try/finally` with explicit cleanup on both paths and avoid higher-order array methods, fixing compilation on older UTS→Swift compilers.
- Android: rework the activity-result callback registration (stable forwarder + replaceable handler), fixing compilation on older Kotlin compilers.

# 1.0.0

- Implemented Android, iOS, and HarmonyOS system document picker APIs, metadata lookup, cache staging, and temporary-file cleanup.
- Added a cross-platform demo page for picker and staging flows.
- Raised the minimum HBuilderX version to 5.26.
