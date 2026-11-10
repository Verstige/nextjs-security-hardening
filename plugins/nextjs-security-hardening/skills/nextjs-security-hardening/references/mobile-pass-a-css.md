# Mobile Pass A — Touch Targets, Safe-Area, 16px Font

Drop-in CSS snippets for making admin/trader pages phone-usable
without redesigning layouts. Tested on iOS Safari + Android Chrome.
Apply these to any admin page CSS file. They're
independent of layout — just paste at the end of the existing
stylesheet.

This is **Pass A** (quick polish), not **Pass B** (mobile-first
redesign). For Pass B you'd want bottom sheets, swipe gestures,
dedicated mobile layouts.

---

## 1. Viewport — enable safe-area

Required once, in the root layout, so `env(safe-area-inset-*)` works
on iPhone X+ (notch + home indicator). File: `src/app/layout.tsx`.

```typescript
export const viewport: Viewport = {
    width: 'device-width',
    initialScale: 1,
    maximumScale: 5,
    userScalable: true,
    viewportFit: 'cover',  // <-- add this
    themeColor: '#050507',
};
```

Without `viewportFit: 'cover'`, iOS still respects safe-area but the
browser doesn't expose it to the page. With it, the page receives
correct insets in `env(safe-area-inset-top | bottom | left | right)`.

---

## 2. Shell padding — bottom safe-area + nav clearance

Every admin page has a `.shell` wrapper. The bottom padding needs to
account for: (a) the bottom nav bar height, (b) iPhone home indicator.
One CSS variable handles both.

```css
@media (max-width: 480px) {
    .trader-shell,
    .at-shell,
    .ats-shell,
    .tp-shell,
    .tp-not-found {
        padding: 16px 16px calc(96px + env(safe-area-inset-bottom, 0px));
    }
}
```

96px is roughly the height of a 56-64px top nav + 32px of breathing room.
Adjust per project.

---

## 3. Buttons — 44px minimum touch target

Apple HIG says 44pt, Material says 48dp. Pick one. The pattern:

```css
@media (max-width: 720px) {
    .my-btn {
        padding: 12px 14px;
        min-height: 44px;          /* Apple HIG */
        font-size: 13px;
        flex: 1 1 calc(50% - 4px); /* 2-col button grid on mobile */
    }
}
```

For primary CTAs (the "Send" / "Save" button on a form), make it
full-width and centered:

```css
@media (max-width: 720px) {
    .my-primary-btn {
        width: 100%;
        padding: 16px 24px;
        font-size: 15px;
        justify-content: center;
    }
}
```

For the secondary action ("Clear", "Cancel"), stack vertically below
the primary with `flex-direction: column-reverse`:

```css
@media (max-width: 720px) {
    .my-form-actions {
        flex-direction: column-reverse;
        gap: 10px;
    }
    .my-form-actions .my-secondary-btn {
        width: 100%;
        justify-content: center;
    }
}
```

---

## 4. Inputs — 16px font prevents iOS zoom

iOS Safari zooms in on any input with `font-size: < 16px` when the
input is focused. On a desktop-styled form, this is jarring — the
page suddenly enlarges and the user has to manually zoom out.
Solution: force 16px on mobile.

```css
@media (max-width: 720px) {
    .my-input,
    .my-textarea,
    .my-select {
        padding: 14px 14px;
        font-size: 16px;  /* prevents iOS zoom-on-focus */
    }
}
```

---

## 5. Form layout — single-column stack on phones

Most admin forms use 2-3 column grids. On phones, stack to 1 column
with consistent vertical rhythm.

```css
@media (max-width: 720px) {
    .my-form-row {
        grid-template-columns: 1fr !important;  /* force override */
        gap: 14px;
    }
}
```

The `!important` is intentional — it overrides any prior breakpoint
that set 2-col. Use it for form-row specifically, not as a general
hammer.

---

## 6. Section borders — drop on phones

Cards with 1px borders look fine on desktop. On phones, the borders
eat the 16px side margin and the card visually merges into the
background. Bleed the card to the edge:

```css
@media (max-width: 720px) {
    .my-section {
        padding: 20px 16px;
        margin-left: -16px;
        margin-right: -16px;
        border-radius: 0;
        border-left: none;
        border-right: none;
    }
}
```

This makes the section a full-bleed "block" on mobile, with the
content padded inside.

---

## 7. Stats grid — collapse to 2x2 or single column

Five-stat horizontal grid becomes unwieldy on phones. Two options:

**Option A: 2x2 + centered last item**
```css
@media (max-width: 720px) {
    .my-stats {
        grid-template-columns: repeat(2, 1fr);
    }
}
```

**Option B: 5-up with smaller numbers** (for dashboards where horizontal
feel matters)
```css
@media (max-width: 720px) {
    .my-stats {
        grid-template-columns: repeat(5, 1fr);
        gap: 4px;
    }
    .my-stat-val { font-size: 14px; }
    .my-stat-label { font-size: 8px; }
}
```

Pick A unless the 5-up horizontal feel is the design intent.

---

## 8. Modal — full-bleed on phones

Modals that work on desktop (max-width: 520px, centered) need to
go full-screen on phones.

```css
@media (max-width: 720px) {
    .my-modal {
        max-width: 100%;
        max-height: 95vh;
        border-radius: 14px 14px 0 0;  /* optional: bottom sheet feel */
    }
    .my-modal-bg {
        align-items: flex-end;  /* anchors modal to bottom */
    }
    .my-modal-head { padding: 14px 16px; }
    .my-modal-head h2 { font-size: 15px; }
    .my-form { padding: 16px; }
}
```

---

## 9. Sticky-bottom widget — for live feeds / persistent actions

Pattern: a widget that's in the document flow on desktop, but
sticks to the bottom of the viewport on mobile (above the home
indicator).

```css
@media (max-width: 720px) {
    .my-sticky-widget {
        position: sticky;
        bottom: calc(env(safe-area-inset-bottom, 0px) + 8px);
        margin-top: 16px;
        margin-bottom: 8px;
        z-index: 5;
        box-shadow: 0 -8px 24px rgba(0, 0, 0, 0.4);
    }
    .my-sticky-widget .my-list {
        max-height: 200px;
        overflow-y: auto;
    }
}
```

⚠️ **`position: sticky` may not work inside a parent with
`overflow: hidden`.** Check the DOM hierarchy. If the parent has
`overflow: hidden` (common in tabbed layouts), the sticky element
won't stick. Fix by removing `overflow: hidden` from the parent on
mobile, or use `position: fixed` with explicit top/bottom values.

---

## 10. Search input — 16px font + visible search button

On desktop, a search input is often just an icon + text. On mobile,
add a visible "Search" button next to the input so users on touch
devices have a clear affordance.

```css
@media (max-width: 720px) {
    .my-search-row {
        flex-direction: row;
    }
    .my-search input {
        font-size: 16px; /* iOS zoom prevention */
    }
    .my-search-btn {
        min-height: 44px;
        flex: 0 0 auto;
    }
}
```

---

## 11. Touch target — 40px minimum for non-primary buttons

44px for primary actions. 40px for secondary (Pause, Edit, Activate,
Delete on a list row). Below 40px the tap is unreliable.

```css
@media (max-width: 720px) {
    .my-row-action-btn {
        min-height: 40px;
        padding: 8px 12px;
        font-size: 12px;
    }
}
```

---

## 12. Icon-only buttons — keep ≥ 40x40 hit area

A 16px icon inside a 24px button looks fine on desktop, but tapping
a 24x24 target is unreliable. Expand the hit area with negative
margin or padding:

```css
.my-icon-btn {
    position: relative;
    padding: 4px;
    min-width: 32px;
    min-height: 32px;
}

@media (max-width: 720px) {
    .my-icon-btn {
        /* visual stays small, hit area grows */
        padding: 12px;
        min-width: 44px;
        min-height: 44px;
    }
}
```

---

## 13. Typography scale — 8pt grid on mobile

Reduce type scale proportionally. Don't just shrink the desktop
scale — that produces tiny unreadable text.

```css
@media (max-width: 480px) {
    .my-h1 { font-size: 24px; }   /* was 32px */
    .my-eyebrow { font-size: 10px; }
    .my-body { font-size: 13px; }
    .my-stat-val { font-size: 18px; }  /* was 24-28px */
    .my-stat-label { font-size: 9px; }
}
```

---

## Test it

After applying:

1. Open in iOS Safari — tap every button, fill every form. Verify
   the page doesn't jump on input focus (16px font), every tap
   registers, no horizontal scroll, content not cut off at the
   home indicator.
2. Open in Android Chrome — same. Note: Android Chrome doesn't
   zoom on inputs <16px the way iOS does, but the bigger font is
   still better for thumb-typing.
3. Test with split-screen / pop-up views on iPad. The
   `safe-area-inset-*` values change. If your layout breaks, you
   have hard-coded values somewhere that should be `auto` or
   `env(safe-area-inset-*)`.

---

## What this does NOT fix

- Sticky-on-overflow-hidden parents (see §9 warning)
- Swipe-to-dismiss modals
- Long-press context menus
- Bottom-sheet pattern for action pickers
- Reduced-motion preferences
- Dark-mode auto-switching

Those need Pass B (mobile-first redesign) or specific
platform-only work.
