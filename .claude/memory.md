# Mathpop Project Memory

## Project Overview
- **What:** Mathpop (mathpop.io) — free gamified math practice web app
- **Stack:** Single-file HTML/JS/CSS app (`index.html`, ~5200 lines), Firebase Auth + Firestore, PayPal SDK
- **Owner:** Lili (The Bahamas)

## Payment Situation
- PayPal is integrated but Lili cannot withdraw funds to a Bahamian bank (no US bank/routing number, Wise unavailable in Bahamas)
- Gamepasses are currently locked with "Coming Soon" while payment processing is sorted out
- Best path forward: Lemon Squeezy (PayPal payout) or Payoneer (physical Mastercard for ATM withdrawals) — Lili signed up for Payoneer, awaiting identity verification
- Live PayPal client ID is in the code; sandbox ID was replaced

## Current Features
- Modes: Multiply, Long Multiply, Short Divide, Long Divide, Remainder Division, Add, Subtract, Word Problems
- Gamepasses: Customization Pass ($2.99 one-time), Premium Pass ($4.99/mo or $49.99/yr) — both locked as "Coming Soon"
- Premium perks: Crown badge, up to 3 equipped titles, upcoming complex math categories
- Themes: 10 light, 10 dark, 15 neon, 15 gradient (fun multi-color combos), custom color wheel, custom gradient maker
- Quest system: 8 normal quests with progress bars, 14+ hidden/secret quests
- Leaderboards, achievements, avatar crop system, rank progression

## Architecture Notes
- `switchScreen(activeId)` / `goHome()` helper for all screen navigation
- `ALL_SCREENS` array: setup-screen, settings-screen, achievement-screen, leaderboard-screen, gamepass-screen
- Gradient themes use rgba() card colors for bleed-through effect
- `gradDirection` (not gradientDirection) is the variable name for gradient direction state
- `QUEST_MODE_MAP` maps quest IDs to stat keys for progress tracking
- Mobile nav (under 600px): hides text labels and username, shows emoji icons only

## Key Bugs Fixed (March 30, 2026 Session)
- Missing `avatarStatus` element crashed showSettings()
- `subscriptionStatus` vs `subStatus` ID mismatch in unlockPremium()
- `gradientDirection` vs `gradDirection` variable name mismatch broke gradients
- Gradient themes didn't call applyThemeVars() (only set body background)
- Quest progress bars gated behind nonexistent `q.repeatable` property
- Nav bar overflow on mobile portrait
- Titles now render as separate horizontal chips, not merged text
- Primary color picker now derives text/muted colors automatically
- Selecting a pre-made theme clears any active gradient

## Subscription Tiers Plan
- **Customization Pass:** $2.99 one-time — neon themes, gradients, color wheel, gradient maker
- **Premium Pass:** $4.99/mo or $49.99/yr — crown badge, up to 3 titles, unlocks: Fractions, Decimals, Percentages, Geometry (area/perimeter), Basic Algebra
- **Premium Plus Pass (FUTURE):** $9.99/mo or $99.99/yr — all Premium features PLUS additional advanced categories (trig, calculus, statistics, etc.) and a special "Premium Platinum" title for the homepage

## Next Steps
- BUILD the first 5 premium categories: Fractions, Decimals, Percentages, Geometry (area/perimeter), Basic Algebra
- Set up proper subscription billing once Payoneer is verified (likely Lemon Squeezy)
- Re-enable gamepasses when payment flow is working
- Later: build Premium Plus Pass with advanced categories and Platinum title
