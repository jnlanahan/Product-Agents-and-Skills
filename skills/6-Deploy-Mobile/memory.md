# Skill Memory — 6-Deploy-Mobile

<!-- One line per lesson, shorthand, max 7. Promote repeats to SKILL.md. -->

- `eas update` (manual AND CI) without `--environment <env>` ships NO baked EXPO_PUBLIC_* vars → app can't auth → demo mode that users read as "app reverted to an old version".
- Phones fetch the LATEST update group on a branch — a bad publish after a good one wins; fix by republishing correctly, never by waiting.
- OTA-on-push GitHub workflow silently fails without the EXPO_TOKEN repo secret — after any push, verify with `gh run list` before telling the user "it'll reach your phone".
- New native modules + `runtimeVersion: {policy: appVersion}` = old installed binaries ACCEPT the OTA, crash on the missing module at launch, and expo-updates silently rolls back → phone shows the old app; a new build + sideload must land BEFORE any OTA referencing the modules.
- Backend deploy (Railway) and OTA (EAS) are independent pipelines — verify each separately: curl a NEW endpoint on prod, and `eas update:list --branch <ch>` for what phones actually fetch.
- Half-configured Sentry (@sentry/react-native plugin, no org/auth) fails the ENTIRE release build at the source-map upload step — set SENTRY_DISABLE_AUTO_UPLOAD=true in the eas.json build profile env until Sentry setup is complete.
