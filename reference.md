# Figma → Android XML — Reference Tables

Conversion tables only. Rules live in `SKILL.md`. Terms: **Hug contents**, **Fill container**, **Fixed** (Figma's resizing modes).

## Contents
1. Containers · 2. Auto layout · 3. Sizing · 4. Constraints · 5. Frame scale & rounding · 6. Typography · 7. Opacity · 8. Shapes & strokes · 9. Gradients · 10. Shadows · 11. Assets · 12. Widgets & states · 13. Fallback names

## 1. Containers
| Design structure | Use |
|---|---|
| Children positioned relative to each other on both axes, or overlapping | `ConstraintLayout` |
| Single-axis stack, no overlap | `LinearLayout` |
| Layers stacked on the same area only | `FrameLayout` |
| Content taller than the screen at 200% font | `NestedScrollView` (one child) |
| Repeated items | `RecyclerView` + item layout |
| Wrapping chips/tags | `ChipGroup` (M2/M3); `FlexboxLayout` only if already a dependency |
| Collapsing header | `CoordinatorLayout` + `AppBarLayout` |

## 2. Auto layout
| Figma | Android |
|---|---|
| Vertical / horizontal | `LinearLayout android:orientation`, or a `ConstraintLayout` chain |
| Padding | `android:paddingStart/End/Top/Bottom` on the container |
| Gap | margin on every item after the first, or `android:showDividers="middle"` + a spacer drawable |
| Space between | chain `spread_inside`, or `Space` with `layout_weight="1"` |
| Alignment | `android:gravity` / `android:layout_gravity` / constraint bias |
| Absolute-positioned child | overlay in `FrameLayout`/`ConstraintLayout` |
| Wrap | `ChipGroup` / `FlexboxLayout` |
| Min / max width | `android:minWidth`/`maxWidth`, or `app:layout_constraintWidth_min`/`_max` |

Put the gap and padding on the element Figma puts them on. Never split one Figma padding across a parent and a child.

## 3. Sizing
| Figma | In `LinearLayout` | In `ConstraintLayout` |
|---|---|---|
| Hug contents | `wrap_content` | `wrap_content` |
| Fill container (main axis) | `0dp` + `layout_weight="1"` | `0dp` + both constraints |
| Fill container (cross axis) | `match_parent` | `0dp` + both constraints |
| Fixed | dp value (icons, avatars, images, fixed-size controls only) | same |
| Aspect ratio locked | — | `0dp` + `app:layout_constraintDimensionRatio="W:H"` |

## 4. Constraints (frames without auto layout)
| Figma | Android |
|---|---|
| Left / Right | constrain start / end |
| Left & Right | `0dp`, both sides constrained |
| Center | both sides constrained, bias 0.5 |
| Scale | `app:layout_constraintWidth_percent` or a percent `Guideline` |

## 5. Frame scale & rounding
Figma values are in frame units. Convert to dp with the first matching row:

| Frame | dp = |
|---|---|
| Repo doc defines a scale factor (e.g. sdp projects) | value ÷ that factor (use the doc's rounding) |
| Width 320–480 | value (1:1) |
| Width > 480 and height ÷ width ≥ 1.6 (phone at @2x/@3x/@4x) | value ÷ round(width ÷ 375) — 720→2, 750→2, 1080→3, 1125→3, 1242→3, 1440→4 |
| Width > 480 and height ÷ width < 1.6 (tablet / landscape) | value (1:1); put the layout in the matching `-sw600dp`/`-land` folder |
| Width < 320 (watch, widget) | value (1:1); report it |

Sizes in sdp/ssp projects: `_Nsdp` dimen with N = converted value, rounded per the repo doc; if none, round half down.
Without a scale library: round to whole dp; if within 1dp of a multiple of 4, snap to it.
Screenshot-only: scale = screenshot px width ÷ frame dp width (assume 360dp when unknown).

## 6. Typography
| Figma | Android |
|---|---|
| Size 16 | `android:textSize="16sp"` |
| Line height 24px | `lineHeight="24sp"` |
| Line height 150% (size 16) | 16 × 1.5 = `24sp` |
| Line height Auto | omit |
| Letter spacing −2% | `android:letterSpacing="-0.02"` |
| Letter spacing 0.5px (size 14) | 0.5 ÷ 14 = `0.036` |
| Text case UPPER | `android:textAllCaps="true"` |
| Truncate text | `android:maxLines` + `android:ellipsize="end"` |
| Auto width | `wrap_content` |
| Auto height (fixed width) | width `0dp`/`match_parent`, height `wrap_content` |
| Vertical align | `android:gravity` |

`lineHeight` attribute: minSdk ≥ 28 → `android:lineHeight`; minSdk < 28 → `app:lineHeight` (needs AppCompat/Material inflation).
`android:includeFontPadding="false"` matches Figma metrics more closely. If the app ships Indic, Thai, or Arabic locales, keep the default (true): disabling it can clip glyphs.

## 7. Opacity → ARGB alpha
Formula: hex(round(opacity × 255)).

| % | Hex | % | Hex | % | Hex |
|---|---|---|---|---|---|
| 100 | FF | 60 | 99 | 20 | 33 |
| 87 | DE | 50 | 80 | 12 | 1F |
| 80 | CC | 40 | 66 | 10 | 1A |
| 70 | B3 | 38 | 61 | 8 | 14 |
| — | — | 30 | 4D | 5 | 0D |

## 8. Shapes & strokes
| Figma | Android |
|---|---|
| Rectangle fill / stroke / radius | `<shape>` drawable |
| Card | `MaterialCardView` (`app:cardCornerRadius`, `app:strokeColor`, `app:strokeWidth`) |
| Per-corner radius | `app:shapeAppearanceOverlay` (`cornerSizeTopLeft`…), or `<corners>` with every corner set |
| Pill | `cornerSize` 50% |
| Inside stroke | `<stroke>` or card `strokeWidth` (both draw inside the bounds) |
| Center stroke | same as inside; report the half-stroke difference |
| Outside stroke | grow the view by 2 × stroke width; report it |
| Dashed | `android:dashWidth` + `android:dashGap` (both required) |

`<corners>`: every corner must start with a radius > 1 or none are rounded. To leave a corner square, set `android:radius` > 1, then override each corner, using `0dp` for square ones.

## 9. Gradients
- `<shape>` linear: `android:angle` must be a multiple of 45. 0 = left→right, 90 = bottom→top, 180 = right→left, 270 = top→bottom.
- Any other angle, or more than 3 stops → vector `<gradient>` via `aapt:attr` with `startX/startY/endX/endY`.
- Radial → `android:type="radial"` + `android:gradientRadius` (required with radial).
- Angular / diamond → unsupported: export an image and report it.

## 10. Shadows
| Figma blur | `elevation` / `cardElevation` |
|---|---|
| ≤ 4 | 2dp |
| 5–12 | 4dp |
| 13–24 | 8dp |
| > 24 | 16dp |

- The view needs a background (outline) to cast a shadow.
- Shadow color: `android:outlineSpotShadowColor` / `outlineAmbientShadowColor` (API 28+; skip below).
- Unsupported in XML — report each: spread, inner shadow, layer blur (`RenderEffect`, API 31+, code only), blend modes.

## 11. Assets
| Type | Rule |
|---|---|
| Icons | VectorDrawable at the Figma frame size (e.g. 24×24dp); outline strokes before export |
| Vector limits | no filters, blur, embedded images, text, patterns, or masks other than clip-path → use PNG/WebP |
| Tint | single-color icons tinted via `app:tint`; multi-color art untinted |
| Directional icons | `android:autoMirrored="true"` |
| Raster | WebP; mdpi/hdpi/xhdpi/xxhdpi/xxxhdpi = 1×/1.5×/2×/3×/4×, or one copy in `drawable-nodpi` |
| Remote images | placeholder + `tools:src`; loaded in code |

Image fill → `android:scaleType`: Fill/Crop = `centerCrop`, Fit = `fitCenter`.

## 12. Widgets & states
| Figma | M2 / M3 | Notes |
|---|---|---|
| Button | `MaterialButton` | `app:backgroundTint`, `app:cornerRadius` |
| Button with icon | `MaterialButton` `app:icon`, `app:iconGravity`, `app:iconPadding` | |
| Icon button | Material icon-button style / `ImageButton` | 48dp + `contentDescription` |
| Text field | `TextInputLayout` + `TextInputEditText` | hint on the layout |
| Card | `MaterialCardView` | |
| Chip | `Chip` in `ChipGroup` | |
| Switch | `MaterialSwitch` (M3) / `SwitchMaterial` (M2) | |
| Checkbox / Radio | `MaterialCheckBox` / `MaterialRadioButton` | |
| Top bar | `MaterialToolbar` (+ `AppBarLayout`) | menu in `res/menu` |
| Bottom nav | `BottomNavigationView` + menu | |
| Tabs | `TabLayout` | |
| FAB | `FloatingActionButton` / `ExtendedFloatingActionButton` | |
| Divider | `MaterialDivider`; in lists `MaterialDividerItemDecoration` | insets via `dividerInsetStart/End` |
| Progress | `LinearProgressIndicator` / `CircularProgressIndicator` | |
| Slider | `Slider` | |

AppCompat-only projects: `AppCompatButton`, `AppCompatEditText`, `SwitchCompat`, `AppCompatCheckBox`, `Toolbar`, a `View` divider.

**States:** variants → `res/color/*.xml` ColorStateList or `<selector>`. Clickable custom views → `?attr/selectableItemBackground`. Disabled default: content 38% opacity, container 12%.

## 13. Fallback names
Use only when Step 0 found no convention.
- Layouts: `activity_*`, `fragment_*`, `item_*`, `view_*`, `dialog_*`, `bottom_sheet_*`
- IDs: `text_*`, `image_*`, `button_*`, `input_*`, `recycler_*`, `card_*`
- Drawables: `ic_*`, `img_*`, `bg_*`, `ill_*`, `selector_*`
- Strings: `<screen>_<element>` (e.g. `login_button_submit`)
- Dimens: `spacing_8`, `spacing_16`, `radius_card`, `icon_size_medium`
