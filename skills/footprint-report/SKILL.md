---
name: footprint-report
description: Display CO2 and water footprint report for Claude Code sessions
---

Run the following bash script exactly as written and present the output to the user. Do not paraphrase or reformat the results.

```bash
#!/usr/bin/env bash

# Force C locale: comma-decimal locales (de_DE, fr_FR) make awk mis-parse
# "431.7045" as 431 and print "431,0" instead of "431.7"
export LC_ALL=C

DB_PATH="${HOME}/.claude/claude-carbon/carbon.db"

if [ ! -f "$DB_PATH" ]; then
  echo "Database not found. Run setup.sh first:"
  echo "  bash ~/code/claude-footprint/scripts/setup.sh"
  exit 1
fi

CURRENT_YEAR="$(date +%Y)"
TODAY="$(date +%Y-%m-%d)"

# Ensure the excluded + water_liters columns exist on pre-existing DBs (idempotent).
# Excluded sessions (non-Anthropic models, e.g. local models) are left out of all aggregates.
sqlite3 "$DB_PATH" "ALTER TABLE sessions ADD COLUMN excluded INTEGER DEFAULT 0;" 2>/dev/null || true
sqlite3 "$DB_PATH" "ALTER TABLE sessions ADD COLUMN water_liters REAL;" 2>/dev/null || true
NOT_EXCLUDED="COALESCE(excluded, 0) = 0"

# --- Aggregates ---
TODAY_CO2="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(co2_grams), 0) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${TODAY}%';" | awk '{printf "%.1f", $1}')"
TODAY_WATER="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(water_liters), 0) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${TODAY}%';" | awk '{printf "%.2f", $1}')"
TODAY_SESSIONS="$(sqlite3 "$DB_PATH" "SELECT COUNT(*) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${TODAY}%';")"

YEAR_CO2="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(co2_grams), 0) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${CURRENT_YEAR}%';" | awk '{printf "%.1f", $1}')"
YEAR_WATER="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(water_liters), 0) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${CURRENT_YEAR}%';" | awk '{printf "%.2f", $1}')"
YEAR_SESSIONS="$(sqlite3 "$DB_PATH" "SELECT COUNT(*) FROM sessions WHERE ${NOT_EXCLUDED} AND started_at LIKE '${CURRENT_YEAR}%';")"

ALL_CO2="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(co2_grams), 0) FROM sessions WHERE ${NOT_EXCLUDED};" | awk '{printf "%.1f", $1}')"
ALL_WATER="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(water_liters), 0) FROM sessions WHERE ${NOT_EXCLUDED};" | awk '{printf "%.2f", $1}')"
ALL_SESSIONS="$(sqlite3 "$DB_PATH" "SELECT COUNT(*) FROM sessions WHERE ${NOT_EXCLUDED};")"
ALL_COST="$(sqlite3 "$DB_PATH" "SELECT COALESCE(SUM(cost_usd), 0) FROM sessions WHERE ${NOT_EXCLUDED};" | awk '{printf "%.2f", $1}')"

# --- CO2 equivalences (all-time total) ---
KM_CAR="$(echo "$ALL_CO2" | awk '{printf "%.0f", $1 / 120}')"
GOOGLE="$(echo "$ALL_CO2" | awk '{printf "%.0f", $1 / 0.2}')"
KM_TGV="$(echo "$ALL_CO2" | awk '{printf "%.0f", $1 / 2.4}')"

# --- Water equivalences (all-time total) ---
BOTTLES="$(echo "$ALL_WATER" | awk '{printf "%.0f", $1 / 0.5}')"
SHOWERS="$(echo "$ALL_WATER" | awk '{printf "%.1f", $1 / 65}')"

# --- Top 5 sessions by CO2 ---
TOP5="$(sqlite3 -separator '|' "$DB_PATH" \
  "SELECT DATE(started_at), project, ROUND(co2_grams, 2), ROUND(COALESCE(water_liters,0), 3), model, ROUND(cost_usd, 4)
   FROM sessions
   WHERE ${NOT_EXCLUDED}
   ORDER BY co2_grams DESC
   LIMIT 5;")"

# --- By project ---
BY_PROJECT="$(sqlite3 -separator '|' "$DB_PATH" \
  "SELECT project, ROUND(SUM(co2_grams), 2), ROUND(SUM(COALESCE(water_liters,0)), 3), COUNT(*), ROUND(SUM(cost_usd), 4)
   FROM sessions
   WHERE ${NOT_EXCLUDED}
   GROUP BY project
   ORDER BY SUM(co2_grams) DESC;")"

echo "==============================="
echo "  claude-footprint report"
echo "==============================="
echo ""
echo "Today (${TODAY})"
echo "  CO2       : ${TODAY_CO2}g"
echo "  Water     : ${TODAY_WATER}L"
echo "  Sessions  : ${TODAY_SESSIONS}"
echo ""
echo "${CURRENT_YEAR}"
echo "  CO2       : ${YEAR_CO2}g"
echo "  Water     : ${YEAR_WATER}L"
echo "  Sessions  : ${YEAR_SESSIONS}"
echo ""
echo "All time"
echo "  CO2       : ${ALL_CO2}g"
echo "  Water     : ${ALL_WATER}L"
echo "  Sessions  : ${ALL_SESSIONS}"
echo "  Cost      : \$${ALL_COST}"
echo ""
echo "--- CO2 equivalences (all-time) ---"
echo "  ${KM_CAR} km by car            (120 gCO2e/km)"
echo "  ${GOOGLE} Google searches       (0.2 gCO2e)"
echo "  ${KM_TGV} km by train           (2.4 gCO2e/km)"
echo ""
echo "--- Water equivalences (all-time) ---"
echo "  ${BOTTLES} water bottles          (0.5 L)"
echo "  ${SHOWERS} showers                (65 L)"
echo ""
echo "--- Top 5 sessions by CO2 ---"
echo "Date        | Project                 | CO2 (g) | Water(L)| Model                          | Cost"
echo "------------|-------------------------|---------|---------|--------------------------------|--------"
while IFS='|' read -r date project co2 water model cost; do
  printf "%-11s | %-23s | %-7s | %-7s | %-30s | \$%s\n" "$date" "$project" "$co2" "$water" "$model" "$cost"
done <<< "$TOP5"
echo ""
echo "--- By project ---"
echo "Project                  | CO2 (g)  | Water(L) | Sessions | Cost"
echo "-------------------------|----------|----------|----------|--------"
while IFS='|' read -r project co2 water sessions cost; do
  printf "%-25s | %-8s | %-8s | %-8s | \$%s\n" "$project" "$co2" "$water" "$sessions" "$cost"
done <<< "$BY_PROJECT"
echo ""
```
