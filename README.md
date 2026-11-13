# ShopLite 🛍️

A Flutter + Supabase Android app that lets users browse products from the
FakeStore API, search and filter them, and save favourites that persist across
devices via Supabase Row Level Security.

---

## Screenshots

| Splash | Login | Sign Up |
|--------|-------|---------|
| _(add screenshot)_ | _(add screenshot)_ | _(add screenshot)_ |

| Home | Products | Product Detail |
|------|----------|----------------|
| _(add screenshot)_ | _(add screenshot)_ | _(add screenshot)_ |

| Favourites | Error state | Empty favourites |
|------------|-------------|-----------------|
| _(add screenshot)_ | _(add screenshot)_ | _(add screenshot)_ |

---

## Setup

### 1. Install Flutter
Follow the [Flutter installation guide](https://docs.flutter.dev/get-started/install).
Requires Flutter stable ≥ 3.35 and Dart ≥ 3.9.

### 2. Create a Supabase project
- Go to [supabase.com](https://supabase.com) → New Project.
- Under **Authentication → Providers**, enable **Email** (email + password).
- Optionally disable **Email confirmation** in Auth settings for local dev.

### 3. Apply the database migration

> **Important – choose ONE method only:**
>
> If your Supabase GitHub integration is configured to run migrations
> automatically on push, do **not** also run `supabase db push`. Running the
> migration twice will cause `CREATE TABLE` to fail because the table already
> exists. Use either the CLI **or** the GitHub integration, never both.

**Using the Supabase CLI (recommended for local dev):**

```bash
# Initialise Supabase in the repo (only needed once)
supabase init

# Link to your hosted project (paste your project ref from the dashboard URL)
supabase link --project-ref YOUR_PROJECT_REF

# Apply the migration – creates the favourites table with RLS policies
supabase db push
```

**Using the GitHub integration only:**
Push the repo. Supabase will detect `supabase/migrations/0001_favourites.sql`
and apply it automatically. Do not run `supabase db push` as well.

### 4. Configure environment variables

```bash
cp env.example.json env.json
```

Open `env.json` and fill in your values from the Supabase dashboard
(Settings → API):

```json
{
  "SUPABASE_URL": "https://xxxx.supabase.co",
  "SUPABASE_ANON_KEY": "eyJ..."
}
```

> `env.json` is listed in `.gitignore` and **must never be committed**.

### 5. Install Flutter packages

```bash
flutter pub get
```

---

## GitHub Setup

### Set repository secrets (run each command and paste the value at the prompt)

```bash
gh secret set SUPABASE_URL
gh secret set SUPABASE_ANON_KEY
```

The `gh` command reads the value interactively — the secret is **never**
written to a file or visible in your terminal history.

### Where to find the built APK

After every push to `main` (or any branch), GitHub Actions runs the workflow.
Navigate to:
**GitHub → Actions tab → the latest run → Artifacts section**

The APK is named `shoplite-<short-SHA>-apk` and is retained for 14 days.

---

## Running Locally

```bash
flutter run --dart-define-from-file=env.json
```

## Building a Release APK Locally

```bash
flutter build apk --release --dart-define-from-file=env.json
```

The APK will be at `build/app/outputs/flutter-apk/app-release.apk`.

---

## Architecture

```
lib/
  main.dart               Entry point; initialises Supabase and ProviderScope.
  app/
    app.dart              Root MaterialApp.router; creates the GoRouter.
    router.dart           All named routes + auth redirect guard.
    theme.dart            Light and dark Material 3 themes.
  core/
    config/env.dart       Reads SUPABASE_URL and SUPABASE_ANON_KEY at build time.
    utils/validators.dart Pure functions for form field validation.
    errors/app_exception  Single exception type with a user-readable message.
  data/
    models/product.dart   Product data class with fromJson / toJson.
    services/
      auth_service.dart        Sign up, sign in, sign out via Supabase Auth.
      product_api_service.dart HTTP calls to the FakeStore API.
      favourites_service.dart  CRUD on the Supabase favourites table.
  providers/
    auth_provider.dart       Riverpod providers for auth state.
    products_provider.dart   Products and categories from the API.
    favourites_provider.dart Favourite IDs; add/remove with optimistic updates.
  screens/
    splash/        2-second splash → Home or Login.
    auth/          Login and Sign-up screens.
    home/          Dashboard with summary cards and nav buttons.
    products/      Searchable, filterable product grid + detail.
    favourites/    User's saved products from Supabase.
  widgets/         Reusable UI components.
```

**Why each layer exists:**

| Layer | Purpose |
|-------|---------|
| Screens | Display data; call provider methods. No business logic. |
| Providers | Bridge between services and the UI using `AsyncValue`. |
| Services | All network/database calls. Convert errors to `AppException`. |
| Models | Strongly typed data structures. Parse and validate JSON. |
| Supabase | Auth + PostgreSQL database with RLS enforced server-side. |

---

## Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `flutter_riverpod` | ^2.6.1 | State management with `AsyncValue` |
| `go_router` | ^14.8.1 | Declarative navigation + auth redirect guard |
| `supabase_flutter` | ^2.9.0 | Supabase Auth and database client |
| `http` | ^1.4.0 | HTTP client for the FakeStore API |
| `google_fonts` | ^6.2.1 | Outfit + Inter typography |

---

## API and Backend

### FakeStore API endpoints

| Method | URL | Description |
|--------|-----|-------------|
| GET | `https://fakestoreapi.com/products` | All products |
| GET | `https://fakestoreapi.com/products/categories` | Category list |
| GET | `https://fakestoreapi.com/products/{id}` | Single product |

No API key is required. All calls have a 10-second timeout.

### Supabase `favourites` table

```sql
create table public.favourites (
  id         uuid        primary key default gen_random_uuid(),
  user_id    uuid        not null references auth.users(id) on delete cascade,
  product_id integer     not null,
  created_at timestamptz not null default now(),
  unique (user_id, product_id)
);
```

**Row Level Security policies:**

| Policy | Operation | Rule |
|--------|-----------|------|
| Users read own favourites | SELECT | `auth.uid() = user_id` |
| Users add own favourites | INSERT | `auth.uid() = user_id` |
| Users delete own favourites | DELETE | `auth.uid() = user_id` |

Every database operation is performed with the **anon key** and enforced by
RLS. The `service_role` key is never used.

---

## Authentication

ShopLite uses Supabase email + password authentication.

- **Sign up**: stores the user's display name in `raw_user_meta_data`.
- **Email confirmation**: if enabled on your project, sign-up returns no
  session. The app shows a dialog instructing the user to check their inbox
  and then navigates to the Login screen. Once the email is confirmed, the
  user can log in normally.
- **Session persistence**: Supabase Flutter stores the session in
  `SharedPreferences`. The user stays logged in across app restarts.
- **Auth redirect guard**: `go_router`'s `redirect` callback re-runs on every
  auth state change, automatically navigating between `/login` and `/home`.

---

## Known Limitations

1. **Forked pull requests** do not receive repository secrets in GitHub
   Actions (this is a GitHub security restriction). The `Validate required
   secrets` step will fail with a clear error message. This is expected
   behaviour and does not affect pushes from the main repository.

2. **Anon key inside the APK**: `SUPABASE_ANON_KEY` is compiled into the
   release APK via `--dart-define`. The key is designed to be public — it is
   not a secret. Row Level Security policies on the database ensure that each
   user can only read and write their own data, so exposing the key does not
   grant access to other users' data.

3. **What I would add with more time**:
   - Offline caching of products using `drift` or `hive`.
   - Push notifications for price drops using Supabase Realtime.
   - A product detail share button using the `share_plus` package.
   - Integration tests with `integration_test` and a mocked Supabase.
   - Pagination / infinite scroll for the product list.

---

## AI Tools Used

- **Google Gemini (Antigravity)**: Generated all initial drafts of every
  Dart file, the SQL migration, the GitHub Actions workflow YAML, and this
  README. Each file was reviewed for correctness and adjusted to fix
  `flutter analyze` errors (e.g. `CardTheme → CardThemeData`,
  `anonKey → publishableKey`, `Ref → WidgetRef`, missing `dart:async` import).
  The test files were written by the AI and verified by running
  `flutter test` to confirm all 28 tests pass.
#   S h o p L i t e _ A p p  
 