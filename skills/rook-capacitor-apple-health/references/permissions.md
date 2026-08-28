# Availability and permissions — Capacitor · Apple Health

## Contents

- [Check availability](#check-availability)
- [Request read permissions](#request-read-permissions)
- [Select permission types](#select-permission-types)
- [Request nutrition write permission](#request-nutrition-write-permission)
- [Open Apple Health settings](#open-apple-health-settings)
- [Interpret the result](#interpret-the-result)

## Check availability

Guard the flow by platform, then use the shared availability bridge. Although the wrapper is named
`RookHealthConnect`, the iOS implementation checks `HKHealthStore.isHealthDataAvailable()`.

```ts
import { Capacitor } from '@capacitor/core';
import { RookHealthConnect } from 'capacitor-rook-sdk';

export async function isAppleHealthAvailable(): Promise<boolean> {
  if (Capacitor.getPlatform() !== 'ios') return false;

  const { result } = await RookHealthConnect.checkAvailability();
  return result === 'INSTALLED';
}
```

Do not request HealthKit permissions or sync when availability is not `INSTALLED`.

## Request read permissions

Initialize ROOK and register the `user_id` first. Ask from a user-initiated screen that explains why each
visible product feature needs Apple Health data.

```ts
import { RookPermissions } from 'capacitor-rook-sdk';

await RookPermissions.requestAppleHealthPermissions({
  types: [
    'stepCount',
    'height',
    'bodyMass',
    'heartRate',
    'heartRateVariabilitySDNN',
    'workout',
    'sleepAnalysis',
    'oxygenSaturation',
  ],
});
```

Passing `{}` or an empty `types` array asks the native SDK for all supported read permissions. Prefer an
explicit, minimal set that matches the product's visible functionality.

## Select permission types

Common mappings:

| Product data | Useful permission types |
| --- | --- |
| Steps and physical activity | `stepCount`, activity time, energy, distance, flights, speed, workout |
| Sleep | `sleepAnalysis`, `sleepApneaEvent`, heart rate, HRV, respiration, oxygen, temperature |
| Body | `height`, `bodyMass`, BMI, body fat, waist, blood pressure, blood glucose, dietary values |
| Heart rate | `heartRate`, `restingHeartRate`, `walkingHeartRateAverage`, `heartRateVariabilitySDNN` |
| Workouts | `workout`, `workoutRoute`, activity-specific power, speed, distance, effort |
| ECG | `electrocardiogram` |

The `AppleHealthPermissionType` union in package 4.1.1 supports:

```text
appleExerciseTime, appleMoveTime, appleStandTime,
basalEnergyBurned, activeEnergyBurned, stepCount,
distanceCycling, distanceWalkingRunning, distanceSwimming, swimmingStrokeCount,
flightsClimbed, walkingSpeed, walkingStepLength, runningPower, runningSpeed,
stairAscentSpeed, cyclingPower, cyclingSpeed, waterTemperature,
height, bodyMass, bodyMassIndex, waistCircumference, bodyFatPercentage,
bodyTemperature, basalBodyTemperature, appleSleepingWristTemperature,
heartRate, restingHeartRate, walkingHeartRateAverage, heartRateVariabilitySDNN,
electrocardiogram, workout, workoutRoute, sleepAnalysis, sleepApneaEvent,
vo2Max, oxygenSaturation, respiratoryRate, uvExposure, biologicalSex, dateOfBirth,
bloodPressureSystolic, bloodPressureDiastolic, bloodGlucose,
dietaryEnergyConsumed, dietaryProtein, dietarySugar, dietaryFatTotal,
dietaryCarbohydrates, dietaryFiber, dietarySodium, dietaryCholesterol,
dietaryBiotin, dietaryCaffeine, dietaryCalcium, dietaryChloride, dietaryChromium,
dietaryCopper, dietaryFatMonounsaturated, dietaryFatPolyunsaturated,
dietaryFatSaturated, dietaryFolate, dietaryIodine, dietaryIron, dietaryMagnesium,
dietaryManganese, dietaryMolybdenum, dietaryNiacin, dietaryPantothenicAcid,
dietaryPhosphorus, dietaryPotassium, dietaryRiboflavin, dietarySelenium,
dietaryThiamin, dietaryVitaminA, dietaryVitaminB12, dietaryVitaminB6,
dietaryVitaminC, dietaryVitaminD, dietaryVitaminE, dietaryVitaminK,
dietaryWater, dietaryZinc, estimatedWorkoutEffortScore, physicalEffort,
workoutEffortScore
```

Only request types supported by the app's minimum iOS version and used by the product.

## Request nutrition write permission

Request dietary write access only before using `RookEvents.writeNutritionEvent`:

```ts
await RookPermissions.requestDietaryWritePermissions();
```

The app must include `NSHealthUpdateUsageDescription` in `Info.plist`.

## Open Apple Health settings

When a user wants to review or change a previous decision, open Apple Health:

```ts
const opened = await RookPermissions.openIOSSettings();
```

iOS generally does not show the same permission prompt again for a type already decided by the user.

## Interpret the result

- Apple intentionally hides whether read permission was denied. A successful permission request means
  the authorization flow completed, not that every requested read type was granted.
- Empty sync results may mean denied access, no samples, delayed provider sync, or locked HealthKit data.
- Users can grant only part of the requested set.
- Do not request permissions automatically on first launch. Explain the value and ask immediately before
  the feature needs access.
- Keep `NSHealthShareUsageDescription`, `NSHealthUpdateUsageDescription`, the requested types, and the
  app's behavior aligned for App Review.
