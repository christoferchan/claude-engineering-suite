---
name: data-seed
description: Data seeding pipeline — detects app data model, generates realistic seed data using AI, hydrates with external APIs (Google Places, Unsplash), ensures content quality and referential integrity, supports bulk seeding with idempotent SQL/JSON fixtures. Use when the user says "seed data", "generate test data", "populate the database", "I need fake data", or wants realistic content for development or demos.
---

# Data Seed Skill

End-to-end data seeding pipeline: Detect → Plan → Generate → Hydrate → Validate → Load.

## When to Use

- User says "seed data", "generate test data", "populate the database"
- User needs realistic content for development, demos, or screenshots
- User is launching a new city/region and needs location data
- User wants to test edge cases with specific data shapes
- After a data model change to populate new tables
- Before App Store screenshots to have polished demo content

## Pipeline Context Protocol

### On Start

1. Read `.claude/pipeline/manifest.md` if it exists — check if this is part of a feature pipeline (e.g., seeding data for a new feature)
2. Read `.claude/pipeline/features/[slug]/spec.md` if it exists — understand what data the new feature needs
3. Read the codebase's data model — types, migrations, seed files

### On Complete

Write report to `.claude/pipeline/features/[slug]/data-seed-report.md`:

```markdown
# Data Seed Report — [date]

## Scope
entities_seeded: [list]
total_records: [count]
cities_covered: [list]

## Data Sources
| Source | Used For | Records |
|--------|----------|---------|
| AI-generated (Claude) | descriptions, names, bios | X |
| Google Places API | photos, addresses, coordinates | Y |
| Unsplash API | hero images | Z |
| Manual fixtures | edge cases, test accounts | W |

## Quality Checks
- [ ] All required fields populated
- [ ] Coordinates within city bounds
- [ ] No duplicate entries
- [ ] Foreign keys valid
- [ ] Categories/vibes consistent with schema
- [ ] Content moderation: [pass/flagged items]

## Files Generated
- [path to SQL file]
- [path to JSON fixtures]

## How to Load
[exact command to run the seed]
```

Update `.claude/pipeline/manifest.md` with seed status.

## Phase 0A: User Interview

### Required Questions

**1. What needs data?**
"What entities need seeding? (spots, events, trips, reviews, users, all of the above)"
→ Determines which tables to populate.

**2. How much?**
"How many records? (minimal: 5-10, realistic: 50-100, stress test: 500+)"
→ Determines generation scale.

**3. For what purpose?**
- Development (needs variety, edge cases)
- Demo / screenshots (needs polished, curated content)
- Load testing (needs volume, can be lower quality)
- Specific feature testing (needs specific data shapes)

**4. Geographic scope:**
"Which cities or regions? (e.g., Tokyo, London, or 'the top 10 cities')"
→ For location-based apps, seed data needs real coordinates and addresses.

**5. External APIs:**
"Do you have API keys for Google Places, Unsplash, or other data sources?"
→ Determines whether to use real photos/addresses or placeholders.

**6. Existing data:**
"Is the database empty or does it have existing data? Should I merge or replace?"
→ Determines ON CONFLICT strategy.

### Store Preferences

Save to `.claude/pipeline/preferences.md`:
```
## Data Seed Preferences
default_scale: realistic
default_cities: [Tokyo, London, NYC, Paris, Barcelona]
google_places_key: [set/not-set]
unsplash_key: [set/not-set]
conflict_strategy: do-nothing
```

## Phase 1: Data Model Detection

Scan the codebase to understand the data model.

### Detect Schema

```bash
# SQL migrations (Supabase, raw SQL)
ls supabase/migrations/*.sql 2>/dev/null | sort
grep -rn "CREATE TABLE" supabase/migrations/ | head -30

# TypeScript types
find src/ -name "*types.ts" -o -name "*Types.ts" | sort
grep -rn "interface\|type " src/features/*/types.ts | head -50

# Prisma schema
cat prisma/schema.prisma 2>/dev/null | head -100

# Drizzle schema
find src/ -name "schema.ts" -path "*/drizzle/*" -o -name "schema.ts" -path "*/db/*" | head -5

# Existing seed files
find . -name "seed*" -not -path "*/node_modules/*" | head -10
ls supabase/seed/ 2>/dev/null
```

### Map Entity Relationships

Build a dependency graph:

```markdown
## Entity Dependency Graph

profiles (no dependencies — seed first)
  ↓
spots (depends on: profiles for created_by)
events (depends on: profiles, spots for venue)
  ↓
trips (depends on: profiles)
trip_items (depends on: trips, spots)
  ↓
reviews (depends on: profiles, spots)
comments (depends on: profiles, reviews)
likes (depends on: profiles, reviews)
  ↓
lists (depends on: profiles)
list_items (depends on: lists, spots)
  ↓
follows (depends on: profiles × 2)
notifications (depends on: profiles, various)
```

This determines seed order — always seed parent entities before children.

### Detect Required Fields & Constraints

```bash
# Find NOT NULL constraints
grep -rn "NOT NULL" supabase/migrations/ | head -30

# Find UNIQUE constraints
grep -rn "UNIQUE" supabase/migrations/ | head -20

# Find CHECK constraints
grep -rn "CHECK" supabase/migrations/ | head -20

# Find enums
grep -rn "CREATE TYPE\|enum " supabase/migrations/ | head -20

# Find default values
grep -rn "DEFAULT" supabase/migrations/ | head -20
```

Present the data model:
```
Detected Data Model:
| Table | Required Fields | Relationships | Unique Constraints |
|-------|----------------|---------------|-------------------|
| spots | name, lat, lng, city | profiles (created_by) | (name, city) |
| events | title, date, spot_id | spots, profiles | — |
| trips | name, user_id | profiles | — |
| reviews | body, rating, user_id, spot_id | profiles, spots | (user_id, spot_id) |
```

## Phase 2: Seed Plan

Before generating data, present a plan:

```markdown
## Seed Plan

### Order (respecting foreign keys)
1. profiles — 10 users (1 test account, 9 realistic)
2. spots — 50 per city × 5 cities = 250 spots
3. events — 20 per city × 5 cities = 100 events
4. trips — 5 per user × 10 users = 50 trips
5. reviews — 3 per spot (avg) = 750 reviews
6. lists — 2 per user = 20 lists
7. follows — random graph, ~30 connections

### Data Shapes
- Normal: 80% of records — all fields populated, typical content
- Minimal: 10% — only required fields, tests empty state handling
- Edge case: 10% — long names, special characters, boundary values

### External Data
- Google Places: fetch real photos + addresses for spots
- Unsplash: hero images for cities
- Coordinates: real lat/lng for each city's neighborhoods
```

Ask user: "Here's the seed plan. Adjust anything?"

## Phase 3: Data Generation

### AI-Generated Content (Claude)

Generate realistic text content using Claude's knowledge:

**Spots:**
```json
{
  "name": "Kissaten Koyama",
  "description": "A third-generation coffee shop in a narrow Shimokitazawa alley. The owner roasts beans on a vintage Fuji Royal machine from the 1970s. Known for their hand-dripped Ethiopian single origin and homemade cheesecake.",
  "city": "Tokyo",
  "neighborhood": "Shimokitazawa",
  "category": "cafe",
  "vibes": ["cozy", "local_character", "creative"],
  "price_level": 2,
  "lat": 35.6613,
  "lng": 139.6680,
  "opening_hours": "8:00-18:00, closed Wednesdays"
}
```

**Quality guidelines for AI-generated content:**
- Names should feel authentic to the city/culture (not generic "Best Coffee Shop")
- Descriptions should be 2-3 sentences, evocative, specific details
- Vibes should match the description (don't tag a dive bar as "romantic")
- Price levels should be realistic for the city
- Categories should match the app's category enum exactly

**Reviews:**
```json
{
  "body": "Came here on a rainy Tuesday and had the place to myself. The Ethiopian pour-over was exceptional — fruity without being sour. The cheesecake is dense in the best way. Cash only, which caught me off guard.",
  "rating": 4,
  "helpful_count": 7
}
```

**Trips:**
```json
{
  "name": "Golden Week in Tokyo",
  "description": "5 days exploring the city beyond the tourist spots",
  "city": "Tokyo",
  "start_date": "2026-05-02",
  "end_date": "2026-05-06",
  "status": "planned"
}
```

### Different Data Shapes

**Minimal (test empty states):**
```json
{
  "name": "Untitled Spot",
  "city": "Tokyo",
  "lat": 35.6762,
  "lng": 139.6503
}
```

**Edge cases (test robustness):**
```json
{
  "name": "Café Résistance — L'Étoile du Nord (日本語テスト)",
  "description": "A very long description that goes well beyond what most spots have, testing how the UI handles overflow in card components, detail views, and list items. This description intentionally exceeds 500 characters to verify truncation logic works correctly across all surfaces where spot descriptions appear, including search results, map popups, share previews, and the detail page itself.",
  "city": "Paris",
  "lat": 48.8566,
  "lng": 2.3522,
  "vibes": ["food_and_drink", "culture", "nightlife", "outdoors", "local_character", "romantic", "creative"]
}
```

**Boundary values:**
- Rating: 1 (minimum), 5 (maximum)
- Name: 1 character, 200 characters
- Coordinates: at city boundaries, at equator (0,0), near date line
- Dates: past, today, far future, null

## Phase 4: External API Hydration

### Google Places API

If the user has a Google Places API key:

```bash
# Search for real places to get photos and addresses
curl -s "https://maps.googleapis.com/maps/api/place/textsearch/json?query=coffee+shop+shimokitazawa+tokyo&key=$GOOGLE_PLACES_API_KEY" | jq '.results[:5]'

# Get place photos
curl -s "https://maps.googleapis.com/maps/api/place/photo?maxwidth=800&photo_reference=$PHOTO_REF&key=$GOOGLE_PLACES_API_KEY" -o photo.jpg
```

**Hydration strategy:**
1. Generate spot names and descriptions with AI
2. Search Google Places for real versions of those spots
3. Pull real photos, addresses, coordinates, opening hours
4. Merge AI descriptions (more evocative) with Google data (accurate details)

### Unsplash API

For hero images, city photos, generic spot images:

```bash
curl -s "https://api.unsplash.com/search/photos?query=tokyo+cafe&per_page=10" \
  -H "Authorization: Client-ID $UNSPLASH_ACCESS_KEY" | jq '.results[:5] | .[].urls.regular'
```

### Country / City Data

For travel-related apps:
- Timezone, currency, language from REST Countries API
- Emergency numbers, tipping customs from curated datasets
- Neighborhood boundaries from OpenStreetMap

### No API Keys? Use Placeholders

If no external APIs available:
- Use placeholder image URLs (picsum.photos, placehold.co)
- Use approximate coordinates from known neighborhoods
- Flag in the report: "Photos are placeholders — run with Google Places API key for real images"

## Phase 5: Content Quality Checks

Before loading, validate everything:

### Required Field Check
```bash
# Ensure no nulls in required fields
cat seed_data.json | jq '[.spots[] | select(.name == null or .city == null or .lat == null or .lng == null)] | length'
```

### Coordinate Validation
```bash
# Ensure coordinates are within city bounds
# Tokyo: lat 35.5-35.8, lng 139.5-139.9
cat seed_data.json | jq '[.spots[] | select(.city == "Tokyo") | select(.lat < 35.5 or .lat > 35.8 or .lng < 139.5 or .lng > 139.9)] | length'
```

### Duplicate Check
```bash
# Check for duplicate names within a city
cat seed_data.json | jq '[.spots[] | {name, city}] | group_by(.name, .city) | map(select(length > 1)) | length'
```

### Category / Vibe Consistency
```bash
# Verify all categories match the app's enum
VALID_CATEGORIES="cafe,restaurant,bar,museum,park,gallery,shop,hotel"
cat seed_data.json | jq -r '.spots[].category' | sort -u | while read cat; do
  echo "$VALID_CATEGORIES" | grep -q "$cat" || echo "INVALID CATEGORY: $cat"
done
```

### Content Moderation
```bash
# Check for potentially problematic content
# Flag anything that might violate App Store guidelines
grep -iE "nsfw|explicit|offensive|violence|hate" seed_data.json && echo "FLAGGED CONTENT FOUND"
```

### Referential Integrity
```bash
# Verify all foreign keys reference existing entities
# All review.spot_id values should be in spots[].id
# All trip_item.trip_id values should be in trips[].id
```

Report:
```
Quality Checks:
[PASS] All required fields populated (250 spots, 100 events, 50 trips)
[PASS] Coordinates within city bounds (5 cities verified)
[PASS] No duplicates
[PASS] All categories match enum
[PASS] All vibes match enum
[PASS] Content moderation: no flagged content
[PASS] Foreign keys valid
[WARN] 3 spots have placeholder images (no Google Places match)
```

## Phase 6: Generate Seed Files

### SQL Format (Supabase / PostgreSQL)

```sql
-- seed_spots.sql
-- Generated by /data-seed on 2026-04-05
-- Safe to re-run: uses ON CONFLICT DO NOTHING

BEGIN;

INSERT INTO spots (id, name, description, city, neighborhood, category, lat, lng, price_level, created_by)
VALUES
  ('uuid-1', 'Kissaten Koyama', 'A third-generation coffee shop...', 'Tokyo', 'Shimokitazawa', 'cafe', 35.6613, 139.6680, 2, 'test-user-uuid'),
  ('uuid-2', 'Bar Trench', 'Speakeasy-style cocktail bar...', 'Tokyo', 'Ebisu', 'bar', 35.6488, 139.7095, 3, 'test-user-uuid')
ON CONFLICT DO NOTHING;

-- Spot vibes (junction table)
INSERT INTO spot_vibes (spot_id, vibe)
VALUES
  ('uuid-1', 'cozy'),
  ('uuid-1', 'local_character'),
  ('uuid-2', 'nightlife'),
  ('uuid-2', 'creative')
ON CONFLICT DO NOTHING;

COMMIT;
```

### JSON Format (for API-based seeding)

```json
{
  "_meta": {
    "generated": "2026-04-05T14:00:00Z",
    "generator": "/data-seed",
    "version": 1,
    "total_records": 450,
    "cities": ["Tokyo", "London", "NYC", "Paris", "Barcelona"]
  },
  "profiles": [...],
  "spots": [...],
  "events": [...],
  "trips": [...],
  "reviews": [...]
}
```

### Seed Script

Generate a script to load the data:

```bash
#!/bin/bash
# seed.sh — Load seed data into Supabase
# Usage: ./seed.sh [local|staging|production]

ENV=${1:-local}

echo "Seeding $ENV..."

# Order matters: parents before children
supabase db execute --file supabase/seed/01_profiles.sql
supabase db execute --file supabase/seed/02_spots.sql
supabase db execute --file supabase/seed/03_events.sql
supabase db execute --file supabase/seed/04_trips.sql
supabase db execute --file supabase/seed/05_reviews.sql
supabase db execute --file supabase/seed/06_lists.sql
supabase db execute --file supabase/seed/07_follows.sql

echo "Seed complete. Loaded X records."
```

## Phase 7: Geographic Seeding

For location-based apps, special handling for geographic data:

### City Setup

For each target city, establish:
- City center coordinates
- Neighborhood list with approximate bounds
- Typical spot density per neighborhood
- Local naming conventions (Japanese for Tokyo, French for Paris)

```markdown
## Tokyo Neighborhoods
| Neighborhood | Center Lat | Center Lng | Radius (km) | Spot Density |
|-------------|-----------|-----------|-------------|-------------|
| Shibuya | 35.6580 | 139.7016 | 1.5 | high |
| Shimokitazawa | 35.6613 | 139.6680 | 0.8 | medium |
| Asakusa | 35.7148 | 139.7967 | 1.0 | high |
| Yanaka | 35.7252 | 139.7672 | 0.5 | low |
```

### Coordinate Generation

Scatter spots realistically within neighborhoods:
- Not in a grid — add random offset
- Not in water or parks (unless the spot IS a park)
- Clustered around main streets (slightly higher density along known corridors)

```javascript
function randomCoordInNeighborhood(center, radiusKm) {
  const r = radiusKm * Math.sqrt(Math.random()) / 111;
  const theta = Math.random() * 2 * Math.PI;
  return {
    lat: center.lat + r * Math.cos(theta),
    lng: center.lng + r * Math.sin(theta) / Math.cos(center.lat * Math.PI / 180)
  };
}
```

## Phase 8: Image Hydration

### Strategy

1. **Real photos** (Google Places API) — best for spots that exist in reality
2. **Stock photos** (Unsplash API) — good for generic city/category images
3. **Placeholder URLs** (picsum.photos) — acceptable for development

### Upload to Storage

If the app uses Supabase Storage or similar:

```bash
# Download image
curl -s -o /tmp/spot-photo.jpg "$IMAGE_URL"

# Upload to Supabase Storage
supabase storage upload spot-photos/[spot-id]/hero.jpg /tmp/spot-photo.jpg

# Or generate the SQL with storage URLs
INSERT INTO spot_photos (spot_id, url, is_hero)
VALUES ('uuid-1', 'https://[project].supabase.co/storage/v1/object/public/spot-photos/uuid-1/hero.jpg', true);
```

## Autonomous Mode

When running autonomously (invoked by /ship or a subagent):

- **Skip interview** if the spec or manifest describes what data is needed
- **Use conservative scale** — seed 10-20 records per entity (enough to test, not overwhelming)
- **Skip external API hydration** if no API keys are configured — use placeholders
- **Always use ON CONFLICT DO NOTHING** — idempotent by default
- **Generate SQL files** but do NOT execute them — write to `supabase/seed/` and report
- **Write data-seed-report.md** to the pipeline feature directory
- **Flag quality issues** but don't block — missing photos are not a blocker

## Anti-Patterns

- DON'T seed production with test data — always confirm the target environment
- DON'T generate UUIDs at random without tracking them — foreign keys need matching IDs
- DON'T ignore character encoding — test with unicode, emoji, RTL text
- DON'T seed without ON CONFLICT — re-running should be safe
- DON'T use sequential IDs for UUIDs — use proper UUID generation
- DON'T generate data that violates the schema — check constraints, enums, NOT NULL
- DON'T seed user passwords in plain text — use hashed values or auth provider IDs
- DON'T skip the dependency graph — seeding reviews before spots will fail on FK constraints
