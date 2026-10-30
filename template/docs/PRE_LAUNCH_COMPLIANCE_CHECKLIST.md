# Pre-launch compliance checklist

A reusable checklist for any app, site, browser extension or product. It is one self-contained file: copy it into any project as it is, with no other file needed.

**When to use it: only at the end of production**, once features are frozen and the product is about to be released (and again before each major release). Not during day-to-day building.

> **Agents:** do not act on this file, plan tasks from it or change the product because of it until the owner says the product is at the end of production and asks for the compliance check. Until then, leave it untouched.

Tick an item only after checking it in the real, shipped product, and write the evidence beside it (a link, a file path, a screenshot path, a date). If an item does not apply, mark it "not applicable" and give the reason.

This is an engineering checklist, not legal advice. Have a qualified person review the legal texts for each place where you operate.

Checklist version 1.1.

## Project details (fill in per project)

| Field | Value |
| ----- | ----- |
| Product name | |
| Platforms (web, browser extension, iOS, Android, email) | |
| Legal business name and registration (organisation) number | |
| VAT / tax number | |
| Registered address, contact email and phone | |
| Countries where it is offered | |
| What is sold (nothing, one-off purchases, subscriptions) | |
| Does it use AI? Which providers? | |
| Can children use it? Minimum age | |
| Data protection contact (and DPO if required) | |
| Where the evidence is kept (folder or doc) | |
| Review date, reviewer and release version | |

## Which rules apply

The regions and features you filled in above decide which items are mandatory. Common triggers:

| Trigger | Rules to check |
| ------- | -------------- |
| Users in the EU, EEA (including Norway) or UK | GDPR / UK GDPR; ePrivacy cookie rules (in Norway the Electronic Communications Act, whose cookie-consent rules apply from 1 January 2025); consumer cancellation rights (14 days in the EU/EEA, Norway's *angrerettloven*) |
| Users in California / the US | CCPA/CPRA ("Do not sell or share" where it applies), COPPA for children under 13, CAN-SPAM for email |
| Marketing email | CAN-SPAM, ePrivacy / national marketing law (opt-in in the EU/EEA, Norway's *markedsføringsloven*), Gmail and Yahoo bulk-sender rules (one-click unsubscribe) |
| Consumer products in the EU | European Accessibility Act (in force since 28 June 2025; micro-enterprises offering services are exempt); target WCAG 2.2 AA |
| AI features | EU AI Act transparency duties (telling people they are dealing with AI, labelling generated content; check the current dates), the AI provider's usage and data terms |
| App stores | App Store privacy label and account-deletion rule; Google Play Data safety form and account-deletion rule; age ratings |
| Browser extension | Chrome Web Store / Firefox Add-ons: privacy fields, "limited use" of user data, single purpose, minimum permissions |

---

## A. Legal documents

### 1. Privacy policy
- [ ] Written in plain language and reachable from the app, the website footer, the sign-up screen, each store listing and the extension listing.
- [ ] Says what personal data is collected, why, the legal basis, who receives it (processors and third parties, including hosting and AI providers), where it is stored and transferred (and the safeguard for transfers outside the EU/EEA), how long it is kept, and the person's rights and how to use them (including complaining to the data protection authority).
- [ ] Matches reality: compare it with the data inventory (item 8) and the SDK audit (item 9).
- [ ] Names the data controller and a contact. Has a "last updated" date and a change history.

### 2. Terms of service
- [ ] Covers who may use the product, account rules, acceptable use, intellectual property, user content, AI-generated output (if any), subscriptions and payment terms, disclaimers and liability limits, termination, governing law and dispute route, and how changes are announced.
- [ ] Accepted at sign-up with a clear action; the accepted version and date are recorded per account.

### 3. Refund policy (if anything is sold)
- [ ] States clearly what can be refunded, the time window, how to ask, and how long it takes. Mentions statutory cancellation and withdrawal rights where they apply (for example 14 days in the EU/EEA, and what happens to that right for digital content once delivery starts).
- [ ] Shown before purchase and in the receipt. Matches the store's own refund rules (App Store, Google Play, payment provider).

### 4. Cookie policy (web)
- [ ] Lists every cookie and similar storage (local storage, session storage, IndexedDB, pixels, fingerprinting) by name, purpose, provider, category (strictly necessary, preferences, analytics, marketing) and lifetime. Strictly necessary items (for example a sign-in session cookie or a saved theme choice) are listed too, even though they need no consent.
- [ ] Generated from a scan of the live site, and re-checked after every new script is added.

### 5. Cookie consent banner (web)
- [ ] Not needed if the product only uses strictly necessary storage: record that decision and the scan that proves it.
- [ ] No non-essential cookie or tracker loads before the person agrees.
- [ ] "Accept" and "Reject" are equally easy and equally prominent on the first layer. Choices are granular (analytics, marketing, and so on).
- [ ] The choice is stored, can be changed or withdrawn later from a visible link, and is respected by every script.
- [ ] Mobile apps: the equivalent is a tracking/analytics consent screen, plus the iOS App Tracking Transparency prompt when tracking across apps.

### 6. Business details
- [ ] Legal name, address, contact email (and phone where required), registration (organisation) number and VAT number are shown in the footer or an About/Imprint page, and in each store listing.
- [ ] A working support contact that someone reads, with a stated response time.

---

## B. Data handling

### 7. Check form consents
- [ ] Every form that collects data says what it is used for and links the privacy policy.
- [ ] Consent boxes are unticked by default, specific (one purpose each) and not bundled with accepting the terms.
- [ ] Marketing consent is separate from service messages. The consent record keeps who, when, what text and which version, and withdrawing is as easy as giving it.

### 8. Do not collect unnecessary data
- [ ] Make a data inventory: each field, why it is needed, the legal basis, where it is stored, who can read it, how long it is kept. In the EU/EEA this doubles as the record of processing activities.
- [ ] Remove fields without a clear purpose. Make optional what is not required. Delete or anonymise data when its purpose ends.
- [ ] Logs, analytics, error reports and AI prompts do not contain personal data they don't need (no passwords, tokens or full message bodies in logs), and have a set retention.
- [ ] Permissions asked by the app or extension (microphone, location, contacts, notifications, site access) are requested only when needed, with an explanation, and the product still works without the optional ones.
- [ ] If the processing is high-risk (large-scale sensitive data, children, profiling), a data protection impact assessment (DPIA) is done.

### 9. Audit third-party SDKs and services
- [ ] List every SDK, script, font host, hosting and database provider, analytics, crash reporter, ad network, payment provider, email sender and AI provider, with what data each receives and where it stores it.
- [ ] Remove any that is not needed. Sign data-processing agreements where required. Check the transfer safeguard for each one outside the EU/EEA.
- [ ] For AI providers: confirm whether prompts and outputs are stored or used for training, and for how long, and say so in the privacy policy.
- [ ] Declare them accurately in the privacy policy, the App Store privacy label, the Google Play Data safety form and the extension store privacy fields.

### 10. Age consent and children's data
- [ ] Decide the minimum age and state it. If children may use the product, or you cannot rule it out, get verifiable parental consent where the law asks for it (for example under 13 in the US; 13 to 16 in the EU/EEA depending on the country, 13 in Norway).
- [ ] Collect the least data possible from children, show no behavioural advertising to them, and describe this in the privacy policy.
- [ ] Store listings carry the correct age rating and "designed for children" settings.

### 11. Data deletion request option
- [ ] In the product, signed-in people can delete their account and data (App Store and Google Play both require this). People without an account can ask through a visible email or form.
- [ ] Deletion also covers processors and third parties, and backups and point-in-time restore copies on a stated schedule (say in the privacy policy how long backups keep data). A confirmation is sent and the time limit is met (for example one month).
- [ ] People can also export their data and correct it where the law gives that right.

### 12. Email: unsubscribe
- [ ] Every marketing email has a working unsubscribe link and the sender's physical address. Bulk senders also send the one-click `List-Unsubscribe` and `List-Unsubscribe-Post` headers.
- [ ] Unsubscribes are honoured within the legal time limit (bulk-sender rules: 2 days; CAN-SPAM: 10 business days) and never re-added.
- [ ] Marketing is sent only to people who opted in. Service emails (receipts, security, password reset) are kept separate and carry no marketing.

---

## C. Honest design and claims

### 13. Remove dark patterns
- [ ] No hidden or pre-ticked options, no confusing double negatives, no guilt-trip wording on the decline button, no countdowns or "only 2 left" that are not real, no forced sign-up to browse, no hard-to-find cancel.
- [ ] Cancelling a subscription is as easy as starting it, and possible in the same place (online if it was bought online).

### 14. Remove hidden fees
- [ ] The full price (taxes, fees, renewal price and renewal date, trial length) is visible before the person commits and again on the confirmation.
- [ ] Free trials say when they convert to paid, what it will cost, and how to cancel.

### 15. Remove fake reviews
- [ ] Every review, rating, testimonial and "users say" quote is real, attributable, and not edited to change meaning. Paid or incentivised reviews are labelled. Negative reviews are not hidden.
- [ ] No invented user counts, logos, press mentions or placeholder testimonials left over from design.

### 16. Remove unsupported claims
- [ ] Each claim in the product, store listing and marketing (such as "secure", "private", "end-to-end encrypted", "GDPR compliant", "AI-powered", "#1", "guaranteed", health or financial benefits) has evidence you can show, or is reworded or removed.
- [ ] Features shown in screenshots and listings exist in the shipped version.
- [ ] AI features: people are told when they are talking to an AI or seeing AI-generated content, and its limits are stated (it can be wrong).

---

## D. Accessibility (target: WCAG 2.2 AA)

### 17. Alt text and labels
- [ ] Every meaningful image has useful alt text; decorative images are hidden from screen readers. Icon-only buttons have accessible names. Form fields have visible labels.
- [ ] Video and audio have captions or a transcript. Check with VoiceOver (iOS/macOS), TalkBack (Android) or NVDA/a screen reader on the web.

### 18. Colour contrast
- [ ] Text contrast is at least 4.5:1 (3:1 for large text), and icons, input borders and focus rings at least 3:1, in light and dark themes and in every state (hover, focus, error; disabled is exempt but must still be readable).
- [ ] Meaning is never carried by colour alone.

### 19. Keyboard and focus navigation
- [ ] Everything works without a mouse or touch: logical tab order, visible focus that is never hidden behind sticky bars, no keyboard traps, skip links on the web, dialogs that trap and restore focus, and Esc to close.
- [ ] Targets are at least 24 × 24 CSS px (WCAG 2.5.8); on touch, main actions are 44 × 44 pt on iOS / 48 dp on Android. Text scales to 200% without losing content. Motion respects the reduce-motion setting.
- [ ] Run an automated check (for example axe) on every page and fix the violations, then do one manual pass with keyboard and screen reader. Publish an accessibility statement if the European Accessibility Act applies.

---

## E. Licences

### 20. Fonts, images, icons, sounds and code
- [ ] Every font, image, icon, illustration, sound, music track and template has a licence that allows your use (commercial, app embedding, web embedding, modification), with the source and licence kept in a file such as `ASSETS_LICENSES.md`.
- [ ] Attribution is shown where the licence demands it. Stock and AI-generated assets follow their provider's terms.
- [ ] Open-source dependencies are listed with their licences (for example from `pnpm licenses list`) and are compatible with how you distribute (for example, copyleft licences in a closed app). Required notices ship with the product.

---

## Final sign-off

- [ ] All items above are ticked with evidence, or marked "not applicable" with the reason.
- [ ] Legal texts were reviewed by a qualified person for each operating region.
- [ ] If the product has a public website, `SEO_AND_DISCOVERABILITY.md` was run too (search visibility is not a legal item, but it belongs to the same launch).
- [ ] A named owner will repeat this checklist before each major release and whenever a new SDK, data field, payment flow or region is added.

| Item | Status | Evidence / note |
| ---- | ------ | --------------- |
| 1 Privacy policy | | |
| 2 Terms of service | | |
| 3 Refund policy | | |
| 4 Cookie policy | | |
| 5 Cookie consent banner | | |
| 6 Business details | | |
| 7 Form consents | | |
| 8 Data minimisation | | |
| 9 Third-party SDK audit | | |
| 10 Age consent | | |
| 11 Data deletion request | | |
| 12 Email unsubscribe | | |
| 13 Dark patterns | | |
| 14 Hidden fees | | |
| 15 Fake reviews | | |
| 16 Unsupported claims | | |
| 17 Alt text | | |
| 18 Colour contrast | | |
| 19 Keyboard navigation | | |
| 20 Licences | | |

---

## Appendix: projects built with the project starter

Skip this appendix in projects that don't use the project starter. In those that do, the file lives at `docs/PRE_LAUNCH_COMPLIANCE_CHECKLIST.md`, and these existing docs already cover parts of the work, so check against them instead of starting from scratch:

| Items | Where to look |
| ----- | ------------- |
| 4, 5 | The starter's own storage is strictly necessary: the sign-in session cookie (`docs/SECURITY.md`) and theme and UI preferences in local storage under `STORAGE_PREFIX` (`src/lib/app.ts`). A banner is only needed once analytics or marketing scripts are added. |
| 8, 9 | Hosting, database, files and logs are on Cloudflare (`docs/BACKEND_CLOUDFLARE.md`); the AI provider is Anthropic (`docs/AI_FEATURES.md`). List both, with their data-processing terms, in the SDK audit. Every setting is in `docs/ENV_AND_SECRETS.md`. |
| 11 | D1 point-in-time restore keeps old data for up to 30 days (7 on the Workers free plan), and the `backups/` exports from `docs/DEPLOY_AND_CI.md` hold personal data: state both in the deletion schedule and delete old exports. |
| 13–16 | Seed and demo data come from real public content and `DEMO_MODE` is off in production (`docs/WORKFLOW.md`); check no demo accounts, sample reviews or placeholder claims reach the live site. |
| 17–19 | `docs/DESIGN_SYSTEM.md` §17 (accessibility checklist) and §23 (UX/UI review checklist) already apply WCAG 2.2 AA; `docs/TESTING.md` has the axe setup. Use their results as evidence. |
| Extension | If the product has one, check `docs/BROWSER_EXTENSION.md` permissions and its privacy note against items 8 and 9 and the store's privacy fields. |
| Who does what | `project-manager` turns this checklist into a release task in `docs/ROADMAP.md` when the owner asks; `backend-developer` (data, deletion, export, email), `developer` and `product-designer` (banner, consents, honest design), `qa-tester` and `ux-reviewer` (accessibility evidence), `devops` (live cookie scan, backups); the owner signs off the legal texts. |
