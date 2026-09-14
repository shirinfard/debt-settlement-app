# بازسازی مالی و تسویه بدهی (Debt Settlement & Multi-Asset Financial Restructuring)

A production-ready Flutter app for managing complex debt-settlement /
financial-restructuring scenarios: multi-currency assets (USD, EUR, Gold),
prioritized creditors, essential expenses, and a real-time deficit/surplus
engine with a step-by-step waterfall settlement simulator.

## Architecture (Clean Architecture, 3 layers)

```
lib/
├── core/                     # Cross-cutting: theme, enums/constants
│   ├── theme/app_theme.dart      Dark cinematic theme + color language
│   └── constants/app_constants.dart  Enums (priority, status, category) + Hive box names
│
├── data/                     # DATA layer — persistence & DTOs
│   ├── models/                    Hive-backed models (InflowModel, DebtModel,
│   │                               ExpenseModel, DailyRatesModel, AssetVaultModel)
│   │                               Each ships a hand-written TypeAdapter, so the
│   │                               project builds with plain `flutter pub get`
│   │                               — no build_runner / codegen step required.
│   ├── local/hive_service.dart    Opens boxes, registers adapters at startup
│   └── repositories/finance_repository.dart
│                                   The ONLY class that talks to Hive. Swap this
│                                   file to move to SQLite/Room/Drift later
│                                   without touching domain or UI code.
│
├── domain/                   # DOMAIN layer — pure business rules, no Flutter imports
│   ├── entities/financial_summary.dart   Plain result objects (unit-testable)
│   └── usecases/
│       ├── deficit_calculator.dart   Computes total inflows/debts/expenses,
│       │                             asset values, and net deficit/surplus
│       │                             under both "keep gold" and "sell gold"
│       │                             scenarios.
│       └── waterfall_simulator.dart  Sorts debts by priority (1=urgent first,
│                                      then largest balance) and allocates
│                                      available cash step by step.
│
└── presentation/             # PRESENTATION layer — Provider + widgets
    ├── providers/finance_provider.dart   ChangeNotifier — single source of
    │                                      truth for the UI; re-runs the
    │                                      calculator on every mutation.
    ├── screens/
    │   ├── dashboard_screen.dart   Home: net result hero, summary grid,
    │   │                            keep/sell gold toggle, waterfall view.
    │   ├── inflows_screen.dart     CRUD for cash inflows/resources.
    │   ├── debts_screen.dart       CRUD for creditors + "record payment".
    │   ├── expenses_screen.dart    CRUD for expenses/purchases.
    │   └── assets_screen.dart      Gold/USD/EUR holdings + daily rate entry.
    └── widgets/
        ├── summary_card.dart       Metric card + Toman number formatter.
        ├── priority_tag.dart       Color-coded priority/status pills.
        └── waterfall_view.dart     Renders WaterfallResult step by step.
```

### Why this split matters
- **Domain has zero Flutter/Hive imports** → `DeficitCalculator` and
  `WaterfallSimulator` can be unit-tested with plain Dart, in milliseconds,
  with no widget tree or storage engine involved.
- **Data layer is swappable** → replacing Hive with Room/SQLite later means
  rewriting `finance_repository.dart` only; nothing above it changes.
- **Presentation only depends on the provider**, never on Hive models
  directly for business logic — screens read `FinancialSummary` /
  `WaterfallResult`, not raw persisted records.

## Core financial logic

`DeficitCalculator.calculate()`:
1. Sums received vs. pending inflows.
2. Sums total debt and *remaining* debt (`amount - paidSoFar`).
3. Sums all expenses and essential-only expenses.
4. Converts gold (grams × rate), USD, and EUR holdings into Tomans using the
   daily rates entered on the Assets screen.
5. Produces **two** net results:
   - `netResultGoldSold`  = liquid cash + gold value − (remaining debt + essential expenses)
   - `netResultGoldKept`  = liquid cash − (remaining debt + essential expenses)

`WaterfallSimulator.run()`:
1. Sorts debts by priority level ascending (1 = most urgent), then by
   remaining balance descending within the same tier.
2. Walks the sorted list, paying each creditor fully while cash lasts, then
   partially, then marking the rest unpaid — recording every step
   (`WaterfallStep`) for the UI to render.

The **Keep/Sell gold switch** on the dashboard toggles which of the two net
results (and which waterfall cash pool) is shown live.

## RTL & Persian UI
- `MaterialApp.locale = fa_IR`, wrapped in `Directionality(TextDirection.rtl)`.
- All screen text is in Persian; number formatting uses `intl`'s
  `NumberFormat.decimalPattern('fa')`.
- **Font**: bundle a Persian-shaping font (e.g. Vazirmatn) under
  `assets/fonts/` and uncomment the `fonts:` section in `pubspec.yaml` — the
  theme already references `fontFamily: 'Vazirmatn'`.

## Color language (dark, cinematic dashboard)
| Color  | Meaning                                  |
|--------|-------------------------------------------|
| Red    | Priority-1 debts, net deficit             |
| Gold   | Gold asset, priority accents               |
| Green  | Inflows received, surplus, "paid" status  |
| Amber  | Pending / partial states                  |
| Blue/Purple | USD / EUR currency values           |

## Getting started

```bash
flutter pub get
flutter run
```

No `build_runner` step is needed — all Hive `TypeAdapter`s are hand-written
in the model files themselves.

## Suggested next steps
- Add `flutter_test` unit tests directly against `DeficitCalculator` and
  `WaterfallSimulator` (they take plain lists — no mocking required).
- Add a historical "daily rates" log (currently only the latest rate is kept)
  if trend charts are wanted — `fl_chart` is already a dependency.
- Add biometric/PIN lock (e.g. `local_auth`) before opening the dashboard,
  since this app holds sensitive financial data.
- Add data export (CSV/PDF) of the debt ledger for legal/accounting use.
