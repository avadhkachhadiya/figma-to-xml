---
name: figma-to-android-xml
description: Converts Figma designs into production Android XML layouts for the View system — layouts, drawables, colors, dimens, strings, styles — then verifies them on a device. Use whenever the user shares a Figma link or node, a design screenshot, or asks to "convert design", "implement screen", "build this layout", "match Figma", "redesign this screen per Figma", or fix spacing, padding, ripples or styling of an Android XML screen against a design. Applies to Android Views and XML (Activities, Fragments, RecyclerView items, dialogs, bottom sheets), not to Jetpack Compose, Flutter, iOS or web — do not use it for @Composable UI.
---

# Figma → Android XML

Turn a design into responsive, theme-aware, accessible Android XML that matches the design to ±1dp and follows the host project's conventions.

**Read `reference.md`** (next to this file) at step 5 of the workflow, before converting any value, and whenever a rule below says "per reference". It holds every conversion table: frame scale, units, sizing, typography, opacity, shapes, gradients, shadows, assets, widgets, fallback names.

**Precedence:** the project's own rule files (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, any design-to-code or scale doc in the repo) > this skill > Material defaults. When a project rule contradicts a rule here, follow the project and say so in the report.

## Step 0 — Detect the project (mandatory, before any code)

Read each value from the project. If it cannot be detected, use the fallback and list it under **Assumptions**.

| Value | Where to read it | Fallback |
|---|---|---|
| minSdk / targetSdk | `app/build.gradle(.kts)` `minSdk`/`targetSdk`, or the version catalog | minSdk 21, targetSdk 35 (strictest: compat attributes, edge-to-edge on) |
| Material version | parent of the app theme in `res/values*/themes.xml`: `Theme.Material3.*` → M3, `Theme.MaterialComponents.*` → M2, `Theme.AppCompat.*` → AppCompat only | M3 if `com.google.android.material` is a dependency; otherwise AppCompat widgets and no Material widgets |
| Binding | `buildFeatures { viewBinding / dataBinding }` | ViewBinding (plain layout root, no `<layout>` tag) |
| ID naming | IDs in the 3 layouts nearest to the screen being built | `<type>_<meaning>` snake_case (reference §13) |
| Resource naming | prefixes/suffixes of existing drawables, colors, strings, dimens, styles | reference §13 |
| Size units | existing layouts: `@dimen/_Nsdp`/`_Nssp` (sdp/ssp library), project `@dimen/*` tokens, or raw `dp`/`sp`; drawables may use a different unit than layouts — copy each separately | raw `dp`/`sp` |
| Scale factor | a repo doc that states it; else derive per reference §5 | reference §5 |
| Tokens | `values/colors.xml` + `values-night/colors.xml`, `dimens.xml`, `styles.xml`/`type.xml`, `themes.xml`, `res/font/` | none; create new tokens with the detected naming |
| Text styles | styles applied to existing `TextView`s (`style=` or `android:textAppearance`) | `TextAppearance.App.*` styles |
| Custom views & shared parts | custom classes used in `res/layout`; `<include>` layouts used by 2+ layouts | none |
| Non-Latin scripts | `values-hi`, `-bn`, `-ta`, `-th`, `-ar`, `-fa`, … folders | assume present (keep `includeFontPadding` default) |
| Device | `adb devices` | none → mark device checks "not verified" in the report |

## Workflow

1. Run Step 0.
2. Fetch the design. **Prefer Framelink**: `mcp__Framelink_Figma_MCP__get_figma_data` for node data, `download_figma_images` for assets — it needs no extra skill. If the link points at a sub-node, also fetch its parent frame. If Day and Night frames exist, fetch both. For the visual target, render the whole frame with `download_figma_images` (`nodeId` only, no `imageRef`, `pngScale` 2–3). Its `localPath` must sit inside the project root: use `build/figma_tmp` and delete it when done.
   **Fallback only when Framelink is unavailable** — official Figma MCP: `get_metadata` on every link → `get_design_context` on the target frame (that server requires loading `figma:figma-design-to-code` first) → `get_variable_defs` → `get_screenshot`. If `get_design_context` is truncated or marked sparse, fetch each visible child node separately.
3. Without Figma access (screenshot only): follow **No Figma access** below.
4. Skip hidden layers and device chrome (status bar, navigation bar, keyboard, device frame). Identify scroll areas, repeated items, overlays, pinned elements, and every state variant.
5. Read `reference.md`. Convert every value (size, padding, gap, radius, text size, line height, letter spacing, opacity) from the node's numbers. Never estimate from the screenshot when a number exists.
6. Map design values to existing tokens per **Design vs. existing project** below. Write a hierarchy plan (one line per container).
7. Compare the design with the current screen. List every control that has no existing behavior and ask the user how to handle it before wiring it. Until answered, give its click site a `// TODO: ` comment.
8. Write XML and resources.
9. Run the **Self-check**. Run **Verify on device**. Write the **Report**.

## Core rules

### Layout
- If children need relative positioning on both axes or overlap → `ConstraintLayout`. If children form a single-axis stack with no overlap → `LinearLayout`. Never use `RelativeLayout` or `AbsoluteLayout`. Keep nesting ≤ 4 levels.
- Give every `ConstraintLayout` child at least one horizontal and one vertical constraint.
- Inside `ConstraintLayout`, use `0dp` with both side constraints; never `match_parent`.
- Give `wrap_content` text in `ConstraintLayout` that can grow `app:layout_constrainedWidth="true"`.
- Map Figma sizing per reference §3: Hug contents → `wrap_content`; Fill container → `0dp`/weight/`match_parent`; Fixed → a dp value, allowed only for icons, avatars, images, and controls with a fixed design size.
- Never set a fixed height on a view that contains text.
- Use `start`/`end`; never `left`/`right`.
- Build repeated items as `RecyclerView` + item layout with `tools:listitem`; never copy items by hand.
- If content can exceed the screen at 200% font scale → wrap it in `NestedScrollView` (one child, `android:fillViewport="true"`). Keep pinned buttons outside the scroll view.
- If targetSdk ≥ 35 (edge-to-edge is enforced; at 36 it cannot be opted out) → never hardcode status/navigation bar heights; apply insets in code and list it under Code follow-ups.
- Place an image behind `wrap_content` content as a `0dp × 0dp` view constrained to that content; never `match_parent`.
- In a horizontal `LinearLayout` with `wrap_content` height, `match_parent`-height children take the height of the tallest non-`match_parent` child; a lone `0dp` `Space` collapses the row to 0.
- **Grouped list card:** put no padding on the card; set `android:clipToOutline="true"`. Each row owns its padding (outer rows include the card's edge gap). Dividers get only a horizontal inset (`MaterialDividerItemDecoration` `dividerInsetStart/End`, or margins on a divider view). This makes ripples and touch areas span the full row.
- Draw a rounded level/progress fill as a sized view with a rounded `<shape>`; never `<clip>`.

### Units & text
- Text size and line height in `sp`; everything else in `dp`; never `px`. If the project uses a scale library (sdp/ssp), use its dimens in layouts in place of `dp`/`sp`.
- Convert every value per reference §5–§6.
- Put text styles in reusable styles. Put `lineHeight` inside a TextAppearance only when text views inflate as `MaterialTextView` (M2/M3 theme); otherwise put it in a view style applied with `style=`.
- If text can overflow → `maxLines` + `android:ellipsize="end"`. Never truncate a label that fits at 100% font scale.
- Set weights with font resources (`@font/…`) or the project's font styles; never `textStyle="bold"` on a custom font.
- If the design's font is missing from the project → use the closest project font and report it. Never add a font without asking.
- On non-AppCompat views (`TextClock`, `Chronometer`, views inside RemoteViews/widgets), `app:fontFamily` is ignored → set `android:fontFamily`.

### Color & effects
- If the design color equals the value of a theme attribute the project already maps (e.g. `colorSurface`) → use `?attr/…`. Otherwise use a semantic `@color/*` with a night value. Never hardcode a hex in a layout.
- Fill opacity → bake into `#AARRGGBB`. Layer opacity → `android:alpha`. Values per reference §7.
- Map shadows per reference §10. Report spread, inner shadow, blur, and blend modes as unsupported in XML.

### Widgets
- If the theme is M2 or M3 → use Material widgets (reference §12). If AppCompat only → use AppCompat/framework widgets; never add the Material library without asking.
- `MaterialButton`: style with `app:backgroundTint`, `app:cornerRadius`, `app:strokeColor`; never `android:background`. Set `android:insetTop`/`insetBottom="0dp"` when the design height excludes the 6dp default insets. Under M2, set `android:textAllCaps="false"` unless the design is uppercase. Set the font via `android:textAppearance` or `android:fontFamily`; a `style=` that only sets a font replaces the button's widget style.
- Rounded or circular images → `ShapeableImageView` + `app:shapeAppearanceOverlay` (M2/M3).
- Icons → VectorDrawable via `app:srcCompat`; tint single-color icons with `app:tint`.
- Give every clickable view a pressed state: a ripple (`?attr/selectableItemBackground` or a `<ripple>` drawable) that covers its full touch area. If the design omits pressed/disabled/error states → use Material defaults and list them under Assumptions.
- Never restyle a shared `<include>` or shared component for one screen; report the mismatch.

### Accessibility
- Touch targets ≥ 48dp × 48dp.
- Meaningful images get `android:contentDescription`; decorative ones get `android:importantForAccessibility="no"`.
- The screen must remain usable at 200% font scale and with 50% longer text.
- Text inputs set `android:inputType`, `android:imeOptions`, and `android:autofillHints` (ignored below API 26).
- Show an empty state for every list that can be empty (no results, no data); never leave a blank or half-drawn container.

### Correctness
- Namespaces: `android:` framework, `app:` AndroidX/Material, `tools:` preview only.
- Never invent attributes. Never use an attribute above minSdk without a fallback.
- Every referenced resource must exist. Put sample data only in `tools:text`/`tools:src`.
- Never add a library without asking.
- After a rename, grep the whole repo for the old name: manifest, `tools:context`, analytics/ad name maps, baseline/startup profiles. Keep analytics and remote-config values unchanged.

### Resources
- Reuse a color only if both its light and night values match the design; otherwise add a new light+night pair.
- Before changing a shared resource, grep its usages. If another screen uses it → add a new resource instead.
- When replacing a resource, check every qualifier folder (`-night`, `-nodpi`, `-vNN`, density folders) for an override of the same name.
- Delete resources the redesign orphaned.

## Situations

### No Figma access (screenshot only)
- Derive the scale from a known element: screenshot width ÷ frame width in dp (reference §5); if unknown, assume a 360dp-wide frame.
- Measure every value, round to the 4dp grid, and mark all values as estimated in the report.
- Reuse existing icons; for missing assets, use a placeholder `tools:src` and list each one.

### Frame width doesn't match a standard width
Apply reference §5 exactly; state the frame width and the divisor used in the report.

### Design vs. existing project
- Spacing/size within ±1dp of an existing token → use the token. Otherwise use the converted value.
- Existing component used by 2+ screens differs from the design → keep the component unchanged; report the deviation.
- Existing component used only by this screen → change it to match the design.
- Design copy has placeholders or typos → keep the existing string; report it.
- Design shows sample data (app name, version, icon) → bind the real value; report it.
- Design color differs from an existing token by any amount → new token pair; never alter the existing one.

### Figma pitfalls
- Icons with stacked duplicate strokes → keep only the top one. Identical Day/Night icons → one vector.
- Broken image-fill crop → use the raw image matched to the screenshot; ship it in `drawable-nodpi`.
- CSS `rotate(90deg)` on an icon → `<group android:rotation="90">` in the vector.
- Before building a dynamic visual, check every state variant and note what moves, not just what recolors.
- Never copy web/React/Tailwind/CSS code from Figma output; extract values only.
- Two layers at the same position: the later sibling draws on top and is the one users see. Render the full screen frame and build from the visible layer, not the first match in the tree.
- Image fills: `scaleMode` FILL → `centerCrop`, FIT → `fitCenter`. A fill with `cropTransform` → download it with `needsCropping` + `cropTransform` so the asset already matches the frame's aspect ratio.
- An `ImageView` set to `match_parent` inside a `wrap_content` `FrameLayout`/`MaterialCardView` reports the bitmap's own height and stretches the card; constrain it to the text content instead (see Layout).

## Verify on device

Skip this section only if Step 0 found no device; then mark each item "not verified".

1. Record the device's current `font_scale`, display density, and night mode first.
2. Install and open the screen (navigate via UI if the activity isn't exported).
3. Screenshot light and dark (`cmd uimode night yes|no`) and every state (empty, loading, error, offline, dynamic levels).
4. Set font scale 2.0 and the largest display size; confirm nothing clips, overlaps, or loses its scroll.
5. Measure spacing from `uiautomator dump` bounds, not screenshots; compare to the converted values (±1dp).
6. Press and hold each list row: the ripple must reach the card edges and the dividers.
7. Restore the values recorded in step 1.

## Self-check (all must pass)
- [ ] Step 0 done; fallbacks listed under Assumptions
- [ ] Every padding, gap, size, radius, and text value converted from design numbers per reference (±1dp)
- [ ] No absolute positions, no fixed-height text views, no `match_parent` inside `ConstraintLayout`
- [ ] Constraints complete; nesting ≤ 4; `start`/`end` only
- [ ] No hardcoded colors, strings, or repeated text styles; existing tokens reused per the rules above
- [ ] Lists use RecyclerView; empty state shown; scroll handled; pinned elements outside the scroll view
- [ ] Grouped list cards unpadded; ripples fill the whole row
- [ ] Attributes valid for widget, namespace, and minSdk; all references exist
- [ ] Touch targets ≥ 48dp; content descriptions set; 200% font scale passes
- [ ] Unwired controls have `// TODO: `; placeholder copy reported, not shipped
- [ ] Shared components untouched; orphaned resources deleted
- [ ] Verified on device in light, dark, every state, and max font/display — or marked "not verified"

## Report

End with these sections; write "None" for an empty one:
**Files** · **Detected config** (values + which were fallbacks) · **Key decisions** · **Assumptions** · **Deviations from design** (with the approximation used) · **Design issues found** (contrast, touch targets, inconsistent spacing, missing states) · **Code follow-ups** (insets, adapters, image loading, listeners) · **Verification** (what was checked on device, or "not verified")
