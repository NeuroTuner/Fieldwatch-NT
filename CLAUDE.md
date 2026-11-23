# Fieldwatch-NT

Fork of OffGridPete/Fieldwatch (passive Wi-Fi/BLE scanner, Kotlin + Jetpack Compose, minSdk 29).
Goal: our own hardened variant, built and signed by us, shaped to the owner's needs.

Security review, findings (with file:line), proposed phases and open decisions:
@docs/SECURITY_REVIEW_HANDOFF.md

## Working rules
- Before designing new features or changing behavior, ask the owner what they want; don't assume.
- Every change must compile and pass `./gradlew testDebugUnitTest` before it's committed. Add unit tests for
  every hardening fix (caps, cache bounds, glob matcher, alert rate limit).
- Privacy defaults favor the user: nothing leaves the phone unless the user turns it on, and upgrades never
  re-enable a setting the user turned off.
- Never commit keystores, passwords or local.properties.
- Upstream's `dist/` APK is not ours; don't ship or update it. Our builds come from our own signing key.
