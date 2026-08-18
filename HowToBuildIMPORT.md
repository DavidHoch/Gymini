# How to Build a Gymini Import

This guide explains how to turn raw gym, food, activity, weight, and deficit records into a complete Gymini master JSON file.

Give this file and your raw tracking data to an LLM. Ask it to return one valid `.json` file that follows this structure exactly. Then open Gymini and use **Settings → Import master / backup**.

> Importing a master file replaces the data currently stored in Gymini. Export a backup first.

## Quick instructions for the LLM

1. Read this entire guide and the user's raw data.
2. Convert the raw data into one complete Gymini master object.
3. Preserve every top-level key shown below, even when its value is empty.
4. Output valid JSON only—no Markdown fences, comments, explanations, or trailing commas.
5. Do not invent missing health, calorie, macro, weight, distance, or training values. Use `0`, `null`, or an empty array when the source does not provide them.

## Core JSON rules

- The root must be one JSON object, not an array.
- Use double quotes around all keys and strings.
- Store all dates internally as `YYYY-MM-DD`, for example `2026-08-14`.
- Raw dates such as `14/8/26`, `14/08/26`, `14/8/2026`, and `14/08/2026` all become `2026-08-14`.
- A two-digit year means `20xx`.
- Log names use the display date `DD/MM/YYYY`, for example `GYM 14/08/2026` or `FOOD 14/08/2026`.
- Every object that has an `id` must receive a short, unique string. Use stable IDs such as `food_chicken_01` or `gym_20260814`; do not reuse an ID for another object.
- Store numeric values as JSON numbers, not strings. Use `0`, not `"0"`.
- Keep names from the raw data. Do not translate or rename exercises, meals, foods, or activities unless the user asks.
- Prefer one workout log, one food log, one weight entry, and one deficit log per date. Merge same-day data when it belongs to the same daily record.
- Recalculate all derived values rather than trusting copied totals.

## Complete empty master

Start with this object and fill it from the raw data:

```json
{
  "headers": [],
  "gymLogs": [],
  "sectionLogs": [],
  "routines": [],
  "activities": [],
  "goals": [],
  "consistRule": {
    "gym": 2,
    "act": 2,
    "restEvery": 4
  },
  "deficitLogs": [],
  "startWeight": null,
  "startDate": null,
  "foods": [],
  "meals": [],
  "foodLogs": [],
  "menus": [],
  "targets": {
    "cal": 0,
    "protein": 0,
    "fat": 0,
    "carbs": 0,
    "sugar": 0
  },
  "weights": [],
  "weightGoal": null,
  "collapse": {},
  "badgeUnlocks": {},
  "settings": {
    "defaultMenuId": null,
    "defaultRoutineId": null,
    "restBurn": 0,
    "theme": "dark"
  }
}
```

## Top-level fields

| Key | Purpose |
|---|---|
| `headers` | The editable routine currently shown in Gym → Current. |
| `gymLogs` | Historical saved workouts. |
| `sectionLogs` | Completed gym checks and activity history, including calories and distance. |
| `routines` | Reusable gym routine templates. |
| `activities` | Reusable activity buttons, such as walking or football. |
| `goals` | Weekly completion goals linked by name to a gym section or activity. |
| `consistRule` | Gym/activity consistency requirements. |
| `deficitLogs` | Saved daily calorie-deficit records. |
| `startWeight` | Starting weight in kilograms, or `null`. |
| `startDate` | Goal start date in ISO format, or `null`. It anchors Gymini's weekly/monthly deficit periods. |
| `foods` | Reusable food library. |
| `meals` | The editable meals currently shown in Food → Today. |
| `foodLogs` | Historical saved food days. |
| `menus` | Reusable meal/menu templates. |
| `targets` | Daily calorie and macro targets. |
| `weights` | Historical weight entries. |
| `weightGoal` | Goal weight in kilograms, or `null`. |
| `collapse` | UI-only collapsed sections. Use `{}` for a new import. |
| `badgeUnlocks` | Earned badge dates. Usually use `{}` and let Gymini calculate them from the imported data. |
| `settings` | Default templates, resting calorie burn, and app theme. |

## Gym data

### Current routine: `headers`

The hierarchy is section → muscle group → exercise.

```json
"headers": [
  {
    "id": "hdr_a_01",
    "name": "A",
    "burn": 250,
    "subs": [
      {
        "id": "sub_chest_01",
        "name": "Chest",
        "exercises": [
          {
            "id": "ex_bench_01",
            "name": "Bench Press",
            "reps": 8,
            "sets": 3,
            "weight": 70,
            "pr": 85,
            "burn": 0
          }
        ]
      }
    ]
  }
]
```

- `burn` on a section is the calories burned by completing the whole section.
- If section `burn` is greater than `0`, it overrides the sum of the exercises' `burn` values.
- `weight` and `pr` are kilograms. Use `weight: 0` for bodyweight when no external weight was recorded.
- `reps`, `sets`, `weight`, `pr`, and `burn` are numbers.

### Reusable routines: `routines`

Each routine contains the same `headers` structure. Give the copied headers, groups, and exercises fresh IDs.

```json
"routines": [
  {
    "id": "routine_ab_01",
    "name": "A/B Plan",
    "headers": []
  }
]
```

If this should be the default routine, set `settings.defaultRoutineId` to its ID. Otherwise use `null`.

### Workout history: `gymLogs`

Each saved day is flattened into `entries`:

```json
"gymLogs": [
  {
    "id": "gym_20260814",
    "date": "2026-08-14",
    "name": "GYM 14/08/2026",
    "entries": [
      {
        "header": "A",
        "sub": "Chest",
        "name": "Bench Press",
        "reps": 8,
        "sets": 3,
        "weight": 70,
        "pr": 85,
        "burn": 0,
        "volume": 1680
      }
    ]
  }
]
```

Calculate every entry's volume as:

`volume = reps × sets × weight`

For the example above: `8 × 3 × 70 = 1680`. Keep separate entries when the same exercise was performed with different set/rep/weight combinations and the raw data makes that distinction clear.

### Gym checks and activities: `sectionLogs`

This array stores completed gym sections and completed activities. It is also used for calorie-burn, distance, visit, goal, consistency, and badge calculations.

Gym completion:

```json
{
  "id": "check_20260814_a",
  "date": "2026-08-14",
  "name": "A",
  "burn": 250
}
```

Activity completion:

```json
{
  "id": "activity_log_20260815_walk",
  "date": "2026-08-15",
  "name": "Walking",
  "burn": 300,
  "custom": true,
  "km": 5
}
```

- Omit `custom`, or leave it false, for a gym check.
- Use `custom: true` for an activity.
- `km` is counted for distance badges only on custom activity logs.
- Gym visits count unique dates found in `gymLogs` or non-custom `sectionLogs`.
- If a gym workout has known calories burned, create a matching non-custom `sectionLogs` record so the burn is included in deficit and badge totals.

### Activity buttons: `activities`

These are reusable templates, not history:

```json
"activities": [
  {
    "id": "activity_walk_01",
    "name": "Walking",
    "burn": 300,
    "km": 5
  }
]
```

### Weekly goals and consistency

```json
"goals": [
  {
    "id": "goal_a_01",
    "name": "A",
    "perWeek": 2
  },
  {
    "id": "goal_walk_01",
    "name": "Walking",
    "perWeek": 2
  }
],
"consistRule": {
  "gym": 2,
  "act": 2,
  "restEvery": 4
}
```

Goal names should exactly match a current gym section name or activity name. `gym` and `act` are required completions per week; `restEvery` is the rest-cycle setting used by the app.

## Food data

Macros always use these keys:

```json
{
  "cal": 0,
  "protein": 0,
  "fat": 0,
  "carbs": 0,
  "sugar": 0
}
```

### Food library: `foods`

Food measured by grams stores nutrition per 100 g:

```json
{
  "id": "food_chicken_01",
  "name": "Chicken breast",
  "mode": "g",
  "cal": 120,
  "protein": 23,
  "fat": 2.6,
  "carbs": 0,
  "sugar": 0
}
```

Food measured by units stores nutrition per unit:

```json
{
  "id": "food_egg_01",
  "name": "Egg",
  "mode": "unit",
  "cal": 78,
  "protein": 6.3,
  "fat": 5.3,
  "carbs": 0.6,
  "sugar": 0.2,
  "unitName": "egg",
  "gramsPerUnit": 50
}
```

- `gramsPerUnit` is optional reference information. Use `0` if unknown.
- Deduplicate foods case-insensitively when the name and nutrition basis clearly describe the same food.
- Do not merge foods that have different nutrition values, preparation methods, brands, or serving bases unless the user confirms they are the same.

### Meals and food items

`meals`, every menu's `meals`, and every food log's `meals` use the same structure.

Gram-based item:

```json
{
  "id": "item_chicken_20260814",
  "foodId": "food_chicken_01",
  "foodName": "Chicken breast",
  "mode": "g",
  "grams": 200,
  "per100": {
    "cal": 120,
    "protein": 23,
    "fat": 2.6,
    "carbs": 0,
    "sugar": 0
  }
}
```

Unit-based item:

```json
{
  "id": "item_eggs_20260814",
  "foodId": "food_egg_01",
  "foodName": "Egg",
  "mode": "unit",
  "qty": 2,
  "unitName": "egg",
  "perUnit": {
    "cal": 78,
    "protein": 6.3,
    "fat": 5.3,
    "carbs": 0.6,
    "sugar": 0.2
  },
  "gramsPerUnit": 50
}
```

A meal wraps its items:

```json
{
  "id": "meal_lunch_20260814",
  "name": "Lunch",
  "items": []
}
```

The nutrition snapshot inside `per100` or `perUnit` is required because Gymini calculates logs from the saved item values, not only from the current food library.

Calculation for a gram item:

`item macro = per100 macro × grams ÷ 100`

Calculation for a unit item:

`item macro = perUnit macro × qty`

### Current day: `meals`

Put meals here only if they should appear immediately in Food → Today. Otherwise use an empty array.

### Food history: `foodLogs`

```json
"foodLogs": [
  {
    "id": "foodlog_20260814",
    "date": "2026-08-14",
    "name": "FOOD 14/08/2026",
    "meals": [],
    "totals": {
      "cal": 0,
      "protein": 0,
      "fat": 0,
      "carbs": 0,
      "sugar": 0
    }
  }
]
```

Replace the empty `meals` array with that day's meals. Recalculate `totals` by summing every item in every meal. Do not copy totals that disagree with the item-level data unless the raw source contains only daily totals and no item details.

### Reusable menus: `menus`

```json
"menus": [
  {
    "id": "menu_regular_01",
    "name": "Regular day",
    "meals": []
  }
]
```

Use the same meal/item schema. If this should be the default menu, set `settings.defaultMenuId` to its ID. Otherwise use `null`.

### Daily nutrition targets: `targets`

```json
"targets": {
  "cal": 2200,
  "protein": 160,
  "fat": 70,
  "carbs": 230,
  "sugar": 50
}
```

Use `0` for a target that is unknown or should be hidden.

## Deficit, weight, and goal data

### Deficit history: `deficitLogs`

```json
{
  "id": "deficit_20260814",
  "date": "2026-08-14",
  "eaten": 1800,
  "burn": 250,
  "rest": 2300,
  "deficit": 750,
  "noFood": false,
  "manual": true,
  "source": "import"
}
```

Calculate:

`deficit = rest + burn - eaten`

- `eaten` is the food log's calorie total for that date.
- `burn` is the sum of `sectionLogs[].burn` for that date.
- `rest` is resting calories burned, normally `settings.restBurn`.
- A positive result is a calorie deficit; a negative result is a surplus.
- Create one deficit log per date only when the source gives enough information to do so reliably.
- `source` is optional. `"import"` is a useful value for generated files.

### Weight history

```json
"startWeight": 78,
"startDate": "2026-08-14",
"weights": [
  {
    "date": "2026-08-14",
    "kg": 78
  },
  {
    "date": "2026-08-18",
    "kg": 77.5
  }
],
"weightGoal": 72
```

- Weight values are kilograms.
- Weight entries do not use IDs.
- Use no more than one weight entry per date.
- `startDate` controls when Food history begins and anchors the weekly/monthly deficit cards on Home.

## Settings and badges

```json
"collapse": {},
"badgeUnlocks": {},
"settings": {
  "defaultMenuId": null,
  "defaultRoutineId": null,
  "restBurn": 2300,
  "theme": "dark"
}
```

- `theme` must be `"dark"` or `"light"`. Dark is the app default.
- `defaultMenuId` must match an existing `menus[].id`, or be `null`.
- `defaultRoutineId` must match an existing `routines[].id`, or be `null`.
- `restBurn` is the user's daily resting calorie burn.
- Use `collapse: {}` unless restoring UI collapse state.
- Use `badgeUnlocks: {}` for a newly generated import. Gymini derives and unlocks badges from imported gym, strength-volume, calorie-burn, distance, and consistency data when the Badges screen opens.

## Mapping raw data into Gymini

| Raw record | Gymini destination |
|---|---|
| Current A/B/PPL plan | `headers`; optionally also a reusable entry in `routines`. |
| Completed strength workout | One `gymLogs` object for the date. |
| Completed gym day or gym calorie burn | Non-custom record in `sectionLogs`. |
| Walk, run, football, cycling, or other activity | `sectionLogs` with `custom: true`; optionally add a reusable `activities` template. |
| Food nutrition definition | `foods`. |
| Food eaten on a date | Item inside a meal inside `foodLogs`. |
| Reusable daily food plan | `menus`. |
| Today's editable food | `meals`. |
| Daily calorie/macro goal | `targets`. |
| Resting calorie burn | `settings.restBurn`. |
| Daily deficit data | `deficitLogs`, after checking the formula. |
| Weigh-in | `weights`. |
| Starting date and weight goal | `startDate`, `startWeight`, and `weightGoal`. |

## Recommended conversion process

1. Parse and normalize all dates.
2. Identify the user's current routine, reusable routines, activities, targets, and goal settings.
3. Build a deduplicated food library.
4. Build food item snapshots and group them into meals and daily food logs.
5. Build workout entries and group them into daily gym logs.
6. Build gym/activity completion records in `sectionLogs` when supported by the raw data.
7. Recalculate workout volume, food totals, activity burn, and deficit values.
8. Sort all historical arrays from oldest to newest by `date`.
9. Check all IDs and references.
10. Insert everything into the complete top-level master object and return valid JSON only.

## Validation checklist

Before returning the file, verify all of the following:

- JSON parsing succeeds.
- Every top-level key from **Complete empty master** exists.
- Every stored date is `YYYY-MM-DD` and is a real calendar date.
- Every `GYM` and `FOOD` log name matches its date in `DD/MM/YYYY`.
- IDs are non-empty and unique.
- All `foodId`, `defaultMenuId`, and `defaultRoutineId` references point to existing objects.
- Every meal has `items: []`, even when empty.
- Every gym log has `entries: []`, even when empty.
- Gram items have `grams` and `per100`.
- Unit items have `qty`, `unitName`, `perUnit`, and `gramsPerUnit`.
- Food-log totals equal the sum of their item snapshots.
- Workout `volume` equals `reps × sets × weight`.
- Deficit equals `rest + burn - eaten`.
- No unknown value was fabricated.
- The final response contains only the master JSON object.

## Ready-to-copy prompt

```text
Create a complete Gymini master import from my raw tracking data.

Read HowToBuildIMPORT.md first and follow its schema exactly. Preserve every required top-level key. Normalize every date to YYYY-MM-DD, use unique IDs, retain my original names, deduplicate only clear duplicates, and recalculate food totals, lifting volume, calorie burn, and deficit values. Do not invent missing values. Use 0, null, or an empty array where appropriate.

Return one valid JSON object only. Do not use Markdown code fences and do not add an explanation. The result must be ready to save as gymini-master-import.json and import through Gymini Settings → Import master / backup.

My raw data begins below:

[PASTE OR ATTACH RAW DATA HERE]
```
