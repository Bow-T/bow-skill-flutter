---
name: bow-web-form
description: Catch the recurring <PROJECT_KEY> web-form bug class (apps/admin + apps/partner, Next.js + React 19) BEFORE QC does — the required-`*`/validation drift, unvalidated file uploads, re-inlined phone/email regexes, all-empty submits, and images that don't preview/persist across steps that Sprint 8 logged ~12 times on the new Partner app. These shipped typecheck-green + `bun test`-green because the missing check was never written, not mistyped. The two apps use DIFFERENT paradigms — admin = AntD `<Form>`/`Form.Item rules` (asterisk + submit gate automatic); partner = manual `useState` + a hand-rolled `errors` map + a pure `validateX()` (asterisk, required-ness and validation declared in THREE separate spots that drift). Use when creating or editing any form/field/modal/upload in apps/admin or apps/partner (create/edit store profile, business profile, product, service, promotion, legal-document upload, sign-up/reset/change-password), when binding a phone/email/money/file input, when a QC bug says "missing asterisk / accepts invalid phone-email / lets me submit empty / upload has no type-size check / image not previewed / saved data lost", or when adding a required field. Pairs with [[parity-with-sibling-surface]] (partner mirrors the mobile user flow), [[bow-read-path]] (the value's read side), [[bow-lean]] (reuse the shared validator) and [[test-driven-development]] (extract a pure validator, unit-test it).
---

# Web-form review — the checks static analysis and `bun test` never see

Every Sprint-8 partner-form bug **typechecked and passed `bun test`** when it
shipped. They are missing *behaviours*, not type errors: a required field with no
`*`, an upload that trusts `accept=`, a submit that fires on an empty form. Before
declaring a web form done, walk the code you touched against these. Each cites the
real Sprint-8 site.

> Know which app you're in first — the failure modes differ:
> - **admin** = Ant Design `<Form form={form} onFinish={…}>` + `Form.Item rules`.
>   The `*` and the submit-gate are **structural** (`onOk={() => form.submit()}` only
>   calls `onFinish` when every rule passes — `users/components/UserEditModal.tsx:53,65`).
>   Keep every required field on `rules:[{ required: true, … }]` and these stay free.
> - **partner** = NO AntD Form, NO react-hook-form, NO zod. Manual `useState` + a
>   hand-rolled `errors` object + an `isValid` gate. Nothing links required-ness →
>   asterisk → validation, so they drift. **All five sections below are partner risks.**

---

## 1. The required `*` must derive from the SAME required-ness the validator enforces
Partner's asterisk is an opt-in prop, declared separately from the validator — the
two drift and QC files the gap (<PROJECT_KEY>-2674, 2752, 2672).

- **Trap:** `components/ui/input.tsx:25` exposes `requiredMark?: boolean`; `:205`
  renders the `*` only if the caller passes it. Required-ness itself lives in a
  different file (`onboarding/onboarding-step1.tsx:101 validateStep1`). So a field can
  be validated-required with **no** visible `*` (e.g. category/sub-category at
  `onboarding-step1.tsx:663,671` have `error=` but no `requiredMark`), or a `*` can
  sit on a field the validator never enforces. One fix even hand-rolled a *separate*
  `FieldLabel` asterisk (`forms/edit-store-profile-modal.tsx:657`, cites <PROJECT_KEY>-2752).
- **Rule:** a field is required in **one** place and the label reads its `*` from
  that same source. When you mark a field `requiredMark`, add it to `validateX()` in
  the same edit (and vice-versa); grep the field name in both the JSX and the
  validator before you're done. In **admin**, express required-ness only through
  `rules:[{required:true}]` so AntD renders the `*` — never a manual label asterisk.

## 2. Validate at the boundary with the SHARED validator — never re-inline a regex
Phone/email/password validators already exist and are libphonenumber-backed; a
re-inlined `includes('@')` check accepts garbage (<PROJECT_KEY>-2742 invalid phone/email
submitted; 2639/2664 country code).

- **Reuse these, don't re-author** ([[bow-lean]]):
  - `apps/partner/src/lib/auth/validation.ts` — `validateEmail` (`:27`),
    `validatePassword` + `PASSWORD_RULES` (`:50/:73`, <PROJECT_KEY>-2725 rules must match across
    Sign-up/Reset/Change), `validatePhoneE164` (`:117`, runs `isValidPhoneNumber` from
    `libphonenumber-js` so too-short numbers fail), `toE164` (`:126`).
  - `apps/partner/src/lib/store-contact-validation.ts` — `isValidStorePhone/Email/Address`
    (used by both onboarding Step 1 **and** `edit-store-profile-modal.tsx:181`; keep
    them sharing it — that IS the create↔edit parity).
  - `packages/shared/src/phone-format.ts` — `formatPhoneInternational` / `toStoredPhone`
    / `splitStoredPhone`, the cross-app phone format (admin + partner).
- **Country default is `+33` (France), picker searchable** — `components/auth/
  country-phone-field.tsx:16` (`countries.find(c => c.code === "FR")`), admin
  `components/forms/PhoneInputField.tsx:37` (`defaultCountry='FR'`). A new phone input
  must default FR, not the library's `+1`.
- **Rule:** the same field's validator is used on **every** flow it appears in
  (sign-up, reset, change-password, edit-profile). Password rules that differ across
  flows (2725) = you inlined instead of importing `PASSWORD_RULES`.

## 3. File/image upload: validate TYPE **and** SIZE at pick/drop time — `accept=` is not a gate
`accept=` is a hint the file dialog can honour and drag-drop bypasses entirely
(<PROJECT_KEY>-2677 legal docs, 2880 product media).

- **The shared validators exist — call them at pick time:**
  `apps/partner/src/lib/partner-uploads.ts` → `validateBrandingImage` (`:32`,
  `ACCEPTED_IMAGE_TYPES` + `MAX_IMAGE_BYTES` 10 MB) and `validateDocumentFile` (`:48`,
  pdf/jpeg/png + `MAX_DOC_BYTES`). Correct consumer: `edit-store-profile-modal.tsx:230
  pickImage()` → `const invalid = validateBrandingImage(file); if (invalid) { setError; return; }`
  then preview.
- **Current hole to imitate the fix for, not the bug:** `forms/create-edit-product.tsx:1816
  handleAddImages` only does `URL.createObjectURL` — **no type/size check** — and the
  drop handler (`:226 onDrop → onAdd(e.dataTransfer.files)`) bypasses the `accept=`
  at `:293`. Route it through `validateBrandingImage` / `product-images.ts:21
  validateImage` at pick **and** drop, not only at persist.
- **Rule:** every upload entry point (button pick **and** drag-drop) runs the shared
  type+size validator before preview; the preview shows the newly-picked file
  (<PROJECT_KEY>-2740) and the selection survives to persist (<PROJECT_KEY>-2676/2743 — logo/cover not
  applied). Don't rely on `accept=`.

## 4. Gate submit with a pure `validateX()` — block the all-empty create
A form that validates only some fields, or only on the server, lets an all-empty
record through (<PROJECT_KEY>-2876 create product with every field empty).

- **The pattern that works** (`forms/create-edit-product.tsx`): a pure `errors` map
  (`:1430`), `const isValid = Object.values(errors).every(v => !v)` (`:1496`), and
  `handleSaveProduct(){ setSubmitted(true); if (!isValid) return; … }` (`:1800`) with
  the button `disabled={(submitted && !isValid) || saving}` (`:2020`). First click on
  an empty form populates every error and blocks persist.
- **Weaker, don't copy:** `edit-store-profile-modal.tsx:157` gates only
  `values.name` in `isValid`; the phone/email/address checks live apart in
  `handleSave` (`:167`) and just `return` early — easy to add a field and forget it.
- **Rule:** validation for a form is one pure `validateX(data): Errors` that covers
  **every** required field (see `onboarding/validateStep1` — "reports every missing
  required field at once"); the submit handler runs it and bails on any error. A draft
  path may skip it deliberately (`:1789 handleSaveDraft`) — that's a conscious choice,
  note it.

## 5. Create ↔ edit ↔ onboarding are the SAME form — share the validator, don't fork it
The same entity (store profile, product) is entered on the onboarding wizard, the
create form, and the edit modal. Three copies of the rules drift, and QC files the
one that's behind.

- **Rule:** extract the entity's validator once (`validateStep1`,
  `store-contact-validation`) and call it from create, edit **and** onboarding. This
  is the same "reuse the reference" discipline as [[parity-with-sibling-surface]] —
  the reference here is the flow that already validates correctly. If edit accepts
  what create rejects, they've forked.

---

## Before you say a web form is done
- [ ] Every required field's `*` and its validation come from the **same** source;
      grep the field in both the JSX and `validateX()` (admin: `rules:[{required}]`).
- [ ] Phone/email/password use the shared validators (`lib/auth/validation.ts`,
      `store-contact-validation.ts`, `packages/shared/phone-format.ts`), not a new
      regex; phone defaults to `+33`.
- [ ] Every upload entry (pick **and** drop) runs `validateBrandingImage` /
      `validateDocumentFile` for type+size; the pick previews and survives to persist.
- [ ] Submit runs one pure `validateX()` covering all required fields; an all-empty
      form cannot persist; button gated on `submitted && !isValid`.
- [ ] create/edit/onboarding of the same entity share ONE validator (no fork).
- [ ] Added/changed a validator → added/updated its colocated `*.test.ts`
      (`bun test src`); e.g. `lib/auth/validation.test.ts`, `partner-uploads.test.ts`,
      `validate-step1.test.ts` ([[test-driven-development]]).
