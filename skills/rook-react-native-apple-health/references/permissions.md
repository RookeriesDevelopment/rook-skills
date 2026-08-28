# Availability and permissions — React Native · Apple Health

## Contents

- [Prerequisites](#prerequisites)
- [Check a permission state](#check-a-permission-state)
- [Request read permissions](#request-read-permissions)
- [Permission types](#permission-types)
- [Nutrition write access](#nutrition-write-access)
- [Open Apple Health settings](#open-apple-health-settings)
- [Interpret permission results](#interpret-permission-results)

## Prerequisites

The unified 4.x hook has no Apple Health availability method. Gate Apple Health UI with
`Platform.OS === 'ios'`, verify the HealthKit capability and purpose strings, and use a physical iPhone when
testing real Apple Watch data.

Call `useRookPermissions()` unconditionally at component scope, wait for `ready`, and request access only after
explaining why the app needs each category.

## Check a permission state

Check one exported `AppleHealthPermission` at a time:

```tsx
import {
  AppleHealthPermission,
  useRookPermissions,
} from 'react-native-rook-sdk';

const { ready, appleHealthHasPermissions } = useRookPermissions();

const status = ready
  ? await appleHealthHasPermissions(AppleHealthPermission.STEP_COUNT)
  : 'PERMISSION_NOT_REQUESTED';
```

The status is one of:

- `PERMISSION_NOT_REQUESTED`
- `PERMISSION_INDETERMINATE`
- `PERMISSION_GRANTED`

This result is an inference supplied by the SDK, not proof that HealthKit granted read access for every sample.

## Request read permissions

Request only the data required by the product:

```tsx
const {
  ready,
  requestAppleHealthPermissions,
} = useRookPermissions();

const requestCoreHealthAccess = async () => {
  if (!ready) return;

  await requestAppleHealthPermissions([
    AppleHealthPermission.SLEEP_ANALYSIS,
    AppleHealthPermission.STEP_COUNT,
    AppleHealthPermission.ACTIVE_ENERGY_BURNED,
    AppleHealthPermission.BASAL_ENERGY_BURNED,
    AppleHealthPermission.HEART_RATE,
    AppleHealthPermission.WORKOUT,
  ]);
};
```

Calling `requestAppleHealthPermissions()` without an array requests every value in the 4.1.1 enum. Avoid that
unless the application genuinely uses and explains all categories.

## Permission types

The 4.1.1 package exports these groups through `AppleHealthPermission`:

- Activity and movement: `APPLE_EXERCISE_TIME`, `APPLE_MOVE_TIME`, `APPLE_STAND_TIME`,
  `BASAL_ENERGY_BURNED`, `ACTIVE_ENERGY_BURNED`, `STEP_COUNT`, `DISTANCE_CYCLING`,
  `DISTANCE_WALKING_RUNNING`, `DISTANCE_SWIMMING`, `SWIMMING_STROKE_COUNT`, `FLIGHTS_CLIMBED`,
  `WALKING_SPEED`, `WALKING_STEP_LENGTH`, `RUNNING_POWER`, `RUNNING_SPEED`, `STAIR_ASCENT_SPEED`,
  `CYCLING_POWER`, `CYCLING_SPEED`, `WATER_TEMPERATURE`, `WORKOUT`, `WORKOUT_ROUTE`,
  `ESTIMATED_WORKOUT_EFFORT_SCORE`, `PHYSICAL_EFFORT`, `WORKOUT_EFFORT_SCORE`.
- Body: `HEIGHT`, `BODY_MASS`, `BODY_MASS_INDEX`, `WAIST_CIRCUMFERENCE`, `BODY_FAT_PERCENTAGE`,
  `BODY_TEMPERATURE`, `BASAL_BODY_TEMPERATURE`, `APPLE_SLEEPING_WRIST_TEMPERATURE`.
- Heart and respiratory: `HEART_RATE`, `RESTING_HEART_RATE`, `WALKING_HEART_RATE_AVERAGE`,
  `HEART_RATE_VARIABILITY_SDNN`, `ELECTROCARDIOGRAM`, `VO2_MAX`, `OXYGEN_SATURATION`,
  `RESPIRATORY_RATE`.
- Sleep and profile: `SLEEP_ANALYSIS`, `SLEEP_APNEA_EVENT`, `UV_EXPOSURE`, `BIOLOGICAL_SEX`,
  `DATE_OF_BIRTH`.
- Clinical measurements: `BLOOD_PRESSURE_SYSTOLIC`, `BLOOD_PRESSURE_DIASTOLIC`, `BLOOD_GLUCOSE`.
- Nutrition: `DIETARY_ENERGY_CONSUMED`, `DIETARY_PROTEIN`, `DIETARY_SUGAR`, `DIETARY_FAT_TOTAL`,
  `DIETARY_CARBOHYDRATES`, `DIETARY_FIBER`, `DIETARY_SODIUM`, `DIETARY_CHOLESTEROL`, plus the
  exported micronutrient, fat subtype, vitamin, water, caffeine, and mineral values.

Use enum members instead of raw strings so TypeScript catches renamed or unsupported values.

## Nutrition write access

Request write permission immediately before the first nutrition-writing workflow, not during unrelated onboarding:

```tsx
const { requestWriteNutritionPermission } = useRookPermissions();

await requestWriteNutritionPermission();
```

Also keep `NSHealthUpdateUsageDescription` in `Info.plist`. See `sync.md` for `writeNutritionData`.

## Open Apple Health settings

When the user wants to review or change prior authorization:

```tsx
const { openAppleHealthSettings } = useRookPermissions();

await openAppleHealthSettings();
```

## Interpret permission results

- A resolved request means the HealthKit authorization flow completed; it does not prove the user granted every
  requested read type.
- HealthKit deliberately makes denied read access look like an empty store. Do not label empty results as a denial.
- Do not repeatedly prompt after denial. Explain how to change access in Apple Health and offer the settings action.
- Register the ROOK user before requesting permissions so subsequent sync work has an owner.
