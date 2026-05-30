# VILFORMAT.md

You are an AI agent that converts keyboard keymaps from other formats into a Vial `.vil` backup/export file on a best-effort basis.

Use an existing `.vil` export from the target keyboard as the schema and geometry template whenever available. A `.vil` file is JSON. Do not emit Markdown, comments, trailing commas, or non-JSON syntax in the final `.vil` output.

## Core objective

Given a source keymap in another format, produce a `.vil` JSON file that mirrors the source keymap’s behavior as closely as the `.vil` structure allows.

Prioritize accurate key placement and layer behavior. Preserve Vial-only structures from the template unless the source keymap gives directly mappable replacement data.

## Ground rules

- Treat the target `.vil` export as the authoritative structure for that keyboard.
- Preserve all top-level fields from the template unless you have a grounded reason to modify them.
- Do not invent firmware capabilities, protocol versions, UIDs, settings IDs, encoder counts, combo counts, tap-dance slots, macro slots, or layout geometry.
- Do not include QMK compile-time configuration fields in `.vil` unless the template already represents them in Vial’s runtime backup structure.
- Do not assume that a QMK `keymap.json` `config` object can be serialized directly into `.vil`; Vial exports runtime backup data, not the full QMK build configuration.
- If a behavior cannot be represented in the observed `.vil` structure, leave the closest safe placeholder and document the limitation outside the JSON if a separate explanation is requested.
- The final `.vil` itself must contain only valid JSON.

## Observed `.vil` top-level structure

The target `.vil` template uses this top-level shape:

```json
{
  "version": 1,
  "uid": 5010774632021243529,
  "layout": [],
  "encoder_layout": [],
  "layout_options": 4,
  "macro": [],
  "vial_protocol": 6,
  "via_protocol": 9,
  "tap_dance": [],
  "combo": [],
  "key_override": [],
  "alt_repeat_key": [],
  "settings": {}
}
```

Preserve the same keys, value types, array lengths, object shapes, and protocol metadata from the template unless replacing a value is directly supported by the source keymap and the template structure.

## Field guidance

### `version`

- Preserve the template value.
- Do not infer or upgrade this number.

### `uid`

- Preserve the template value.
- This identifies the target Vial keyboard definition/export context. Do not generate a new UID.

### `layout`

- This is the main layer keymap.
- In the observed template, `layout` is an array of layers.
- Each layer is a nested row/column matrix, not a flat QMK `LAYOUT(...)` array.
- Empty physical positions are represented by the number `-1`.
- Key positions are represented by keycode strings such as `"KC_Q"`, `"KC_BSPACE"`, `"LT3(KC_TAB)"`, or `"MT(MOD_LCTL,KC_DOWN)"` depending on what the export accepts.
- Preserve the exact number of layers from the template unless the source and template clearly support the same layer count.
- Preserve each layer’s row count, row lengths, and `-1` holes.
- Replace only keycode string positions.
- Never replace a `-1` hole with a keycode.
- Never delete rows or columns.

### `encoder_layout`

- This stores per-layer encoder actions.
- In the observed template, it is an array with one entry per layer.
- Each layer contains encoder entries, and each encoder entry is a two-item array: `[counterclockwise_action, clockwise_action]`.
- Preserve this structure from the template unless the source keymap contains directly mappable encoder behavior.
- Use `"KC_TRNS"` for transparent encoder behavior only where the template/source already uses transparent semantics.
- Use `"KC_NO"` only for disabled/no-action behavior.

### `layout_options`

- Preserve the template value.
- Do not infer layout option bitmasks from a source keymap unless the same value is explicitly known for the same target keyboard.

### `macro`

- This stores Vial macro slots.
- In the observed template, `macro` is an array of fixed slots.
- A populated text macro appears as an array containing an operation such as `["text", "..."]`.
- Empty macro slots are empty arrays: `[]`.
- Preserve the template’s macro slot count.
- Convert source macros only when they can be represented by the observed macro operation format.
- Do not place secrets, passwords, or private values into generated macros unless they are explicitly present in the source keymap and the task requires a faithful backup conversion.
- If the source contains macros that cannot be represented with the observed operation format, leave the relevant slot empty or preserve the template value.

### `vial_protocol` and `via_protocol`

- Preserve the template values.
- Do not infer protocol values from QMK source files.

### `tap_dance`

- This stores Vial tap-dance slots.
- In the observed template, `tap_dance` is a fixed-length array.
- Each slot is a five-item array:
  1. tap action
  2. hold action
  3. double-tap action
  4. tap-hold action
  5. tapping term integer
- Empty/default slots use keycodes such as `"KC_NO"` plus a tapping term integer.
- Preserve the number of slots.
- Convert source tap-dance definitions only when the source clearly maps to these five fields.
- If source tap-dance behavior is more complex than these fields, approximate with the closest available fields and prefer preserving normal tap behavior over complex secondary behavior.

### `combo`

- This stores Vial combo slots.
- In the observed template, `combo` is a fixed-length array.
- Each combo slot is a five-item array:
  1. first combo key
  2. second combo key
  3. third combo key or `"KC_NO"`
  4. fourth combo key or `"KC_NO"`
  5. output key/action
- Empty/default combo slots are all `"KC_NO"`.
- Preserve the number of slots.
- Convert source combos only when all trigger keys and the output action can be represented as keycode strings.
- If a source combo has more trigger keys than the observed slot supports, keep the most essential keys only if a best-effort approximation is acceptable; otherwise leave the slot empty.

### `key_override`

- This stores Vial key override slots.
- In the observed template, `key_override` is a fixed-length array of objects.
- Each object contains:
  - `trigger`
  - `replacement`
  - `layers`
  - `trigger_mods`
  - `negative_mod_mask`
  - `suppressed_mods`
  - `options`
- Preserve the number of slots and object keys.
- Convert key overrides only when the source provides equivalent trigger key, replacement key, layer mask, modifier masks, and options.
- Do not guess numeric bitmasks unless they are explicitly known from the source or template.
- For unused slots, preserve the template’s unused/default object values.

### `alt_repeat_key`

- Preserve the template value unless the source keymap has directly mappable alt-repeat data and the target template demonstrates how to encode it.
- If the template uses an empty array, leave it empty unless grounded replacement data exists.

### `settings`

- This stores Vial runtime settings using numeric string keys.
- In the observed template, `settings` is an object whose keys are strings like `"1"`, `"2"`, etc., and whose values are integers.
- Preserve all numeric setting keys and values unless the source setting is known to map exactly to the same Vial runtime setting ID.
- Do not directly copy QMK `config.features`, `config.rgb_matrix`, `config.tapping`, `config.mouse_key`, `config.oneshot`, `config.layer_lock`, or split transport settings into this object unless the setting ID mapping is known.
- If an observed template contains settings IDs but their meanings are not known, preserve them.

## Converting source keymaps into `layout`

### Use the template geometry

The source keymap may be flat while `.vil` may be nested.

A QMK-style source may look like this:

```json
"layers": [
  ["KC_ESC", "KC_Q", "KC_W", "KC_E"]
]
```

A `.vil` target may look like this:

```json
"layout": [
  [["KC_ESCAPE", "KC_Q"], ["KC_W", -1, "KC_E"]]
]
```

The `.vil` geometry is authoritative. Fill keycode string positions in order only when the physical ordering is known. Preserve `-1` holes.

### Mapping method

1. Load the target `.vil` template.
2. Load the source keymap.
3. Determine the source layer count and key count per layer.
4. Determine the target `.vil` layer count and the number of non-`-1` key positions per layer.
5. Establish a physical-position mapping from the source layout macro to the `.vil` nested matrix.
6. For each source layer:
   - Copy the corresponding target template layer.
   - Replace keycode string cells according to the mapping.
   - Leave all `-1` cells untouched.
7. For target layers not present in the source, preserve the template layer or fill only known key positions with transparent/no-action values if that is explicitly desired.
8. Validate that every target layer still has the same nested shape as the template.

### Physical mapping requirement

Do not assume that a flat source layer and a nested `.vil` layer have the same order unless the template, keyboard definition, or paired example proves it.

For a split keyboard, physical ordering may differ between source and export. The source may list keys row-major across both halves, while `.vil` may group by visual rows, hands, thumb clusters, or include extra positions. Always prefer a known geometry mapping over naïve sequential assignment.

### Handling unmatched key counts

If the source has fewer key positions than the target `.vil` non-hole positions:

- Fill only mapped positions.
- Preserve existing template values for unmapped positions, or use `"KC_NO"` only when intentionally disabling an unmapped key.

If the source has more key positions than the target `.vil` non-hole positions:

- Map all positions that have a known target location.
- Do not append extra keys or alter the target geometry.
- Treat unmapped source keys as unsupported by the target `.vil` geometry.

## Keycode normalization

Normalize common QMK aliases to the style used by the `.vil` export when possible.

Observed export style uses longer canonical names for some keys:

- `KC_ESC` → `KC_ESCAPE`
- `KC_BSPC` → `KC_BSPACE`
- `KC_ENT` → `KC_ENTER`
- `KC_SCLN` → `KC_SCOLON`
- `KC_COMM` → `KC_COMMA`
- `KC_SLSH` → `KC_SLASH`
- `KC_SPC` → `KC_SPACE`
- `KC_GRV` → `KC_GRAVE`
- `KC_PSCR` → `KC_PSCREEN`

Preserve keycodes already written in a form accepted by the template.

Use `"KC_TRNS"` for transparent keys when the target `.vil` uses Vial/VIA transparent semantics. Source aliases such as `"_______"` should generally become `"KC_TRNS"` in `.vil` output.

Use `"KC_NO"` for no key / disabled key when the source explicitly means no action.

Preserve advanced QMK/Vial keycode expressions when they are present in the template or known to be accepted by Vial, including forms such as:

- `MO(2)`
- `LT2(KC_MINUS)`
- `LT3(KC_TAB)`
- `MT(MOD_LCTL,KC_DOWN)`
- `LCTL(KC_BSPACE)`
- `LSFT(KC_ENTER)`
- `OSM(MOD_MEH)`

When converting function-like keycodes, match the formatting style accepted by the target export. Avoid adding spaces inside function arguments if the template does not use spaces.

## Layer behavior

- Preserve numeric layer references exactly when they are meaningful for the target layout.
- `MO(n)` remains a momentary layer switch to layer `n` if that layer exists in the target layer array.
- `LTn(KC_X)` or equivalent layer-tap forms should only be used if that exact syntax is known to be accepted by the target `.vil`/Vial export.
- If the source has named layers, convert names to numeric layer indices according to source layer order.
- Do not include layer names in `.vil` unless the template has a field for them.

## RGB and lighting keys

Runtime RGB configuration from QMK source config should not be copied into `.vil` unless the target `.vil` settings mapping is known.

Keycodes that control RGB behavior may be placed in `layout` normally if they appear in the source keymap, for example:

- `RGB_TOG`
- `RGB_MOD`
- `RGB_RMOD`
- `RGB_VAI`
- `RGB_VAD`
- `RGB_HUI`
- `RGB_HUD`
- `RGB_SAI`
- `RGB_SAD`

Do not infer default animation, hue, saturation, value, or speed from a source QMK `config.rgb_matrix.default` object unless the `.vil` template shows the exact numeric setting IDs for those runtime values.

## Firmware/build configuration

The following QMK source sections are build-time or firmware configuration data and are not directly represented in the observed `.vil` structure:

- `keyboard`
- `keymap`
- `layout`
- `config.features`
- `config.tapping`
- `config.split.transport.sync`
- `config.rgb_matrix.animations`
- `config.rgb_matrix.default`
- `config.mouse_key`
- `config.oneshot`
- `config.layer_lock`

Use these fields only as context for understanding available behavior. Do not serialize them into `.vil` unless there is a known `.vil` field or setting ID for the exact runtime value.

## Validation checklist

Before finalizing a generated `.vil` file:

1. Confirm the file is valid JSON.
2. Confirm all top-level template keys are present.
3. Confirm no extra top-level keys were invented.
4. Confirm `version`, `uid`, `vial_protocol`, and `via_protocol` are preserved from the target template.
5. Confirm every `layout` layer has the same nested row/column shape as the template.
6. Confirm every `-1` hole remains numeric `-1`.
7. Confirm every non-hole key position is a string keycode.
8. Confirm `encoder_layout` has the same layer count and encoder-slot shape as the template.
9. Confirm fixed-slot arrays such as `macro`, `tap_dance`, `combo`, and `key_override` retain their template lengths.
10. Confirm all `key_override` entries retain the same object keys.
11. Confirm `settings` remains an object with string keys and integer values.
12. Confirm source transparent aliases such as `_______` were converted to `KC_TRNS` if placed in `.vil` key positions.
13. Confirm common aliases were normalized to the style used by the `.vil` export.
14. Confirm no QMK-only build config was copied into unsupported `.vil` fields.
15. Confirm the output contains only JSON and no explanatory text.

## Best-effort policy

When exact conversion is impossible:

- Prefer preserving the target `.vil` template over producing invalid or speculative JSON.
- Preserve normal typing keys before advanced behavior.
- Preserve layer access keys before cosmetic/runtime settings.
- Preserve source key positions only when the physical mapping is known or can be confidently derived.
- Leave unsupported Vial feature arrays unchanged rather than fabricating data.
- Keep the generated `.vil` importable even if some source behavior cannot be represented.

