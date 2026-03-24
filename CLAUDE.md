# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
poetry install

# Run the tap (sync mode)
tap-klaviyo --config config.json [--state state.json] [--properties catalog.json]

# Run discover (outputs catalog JSON)
tap-klaviyo --config config.json --discover

# Pipe output to a Singer target
tap-klaviyo --config config.json | target-<name> --config target-config.json
```

## Configuration

`config.json` requires:
- `api_key` — Klaviyo private API key (`pk_...`)
- `start_date` — ISO 8601 datetime string (e.g., `"2017-01-01T00:00:00Z"`)
- `user_agent` — email address (optional but recommended)
- `list_ids` — array of Klaviyo list IDs (required only when syncing `list_members2`)
- `segment_ids` — array of Klaviyo segment IDs (required only when syncing `segment_members`)

## Architecture

This is a [Singer](https://singer.io) tap that extracts data from Klaviyo's API and emits Singer-spec JSON (`SCHEMA`, `RECORD`, `STATE` messages) to stdout.

**Two source files:**
- `tap_klaviyo/__init__.py` — entry point, stream definitions, catalog discovery, sync orchestration
- `tap_klaviyo/utils.py` — HTTP request logic, pagination, data transformation, state management

**Stream types:**
- **Full pulls** (`puller='full'`): `lists2`, `metrics2`, `global_exclusions2`, `list_members2`, `events`, `profiles`, `segment_members` — fetched entirely each run via cursor pagination (`links.next`)
- **Incremental pulls** (`puller='incremental'`): legacy metric event streams (`receive`, `click`, `open`, `bounce`, `unsubscribe`, `mark_as_spam`, `unsub_list`, `subscribe_list`, `update_email_preferences`, `dropped_email`) — bookmarked by `since` timestamp in state

**API version split:** The tap straddles two Klaviyo API generations. Streams ending in `2` (e.g., `lists2`, `list_members2`) use the newer Klaviyo v2024-02-15 REST API (`/api/lists`, `/api/profiles/`, etc.) with Bearer/Klaviyo-API-Key auth headers. Legacy streams used the v1 API — most are commented out and disabled.

**Authentication:** `authed_get()` in `utils.py` selects auth style based on stream name. v2 streams use `Authorization: Klaviyo-API-Key <key>` (or `Bearer` if `refresh_token` is present in config). Legacy streams pass `api_key` as a query param.

**State format:**
```json
{"bookmarks": {"receive": {"since": "2017-04-01T00:00:00Z"}}}
```

**Schema files** live in `tap_klaviyo/schemas/<stream_name>.json`. Legacy v1 schemas (e.g., `lists.json`, `list_members.json`) are retained but their streams are commented out in `FULL_STREAMS`.

**Data transformation:** v2 API responses return JSON:API format (`data[].attributes`). `utils.py` has dedicated transform functions (`transform_events_data`, `transform_profiles_data`, `transform_list_members_data`) that flatten attributes to top-level fields before writing Singer records.
