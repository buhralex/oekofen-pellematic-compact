# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Home Assistant custom integration** for Ökofen Pellematic heating systems. It connects to Ökofen heaters via their local TCP/JSON interface and automatically discovers all available components (heating circuits, hot water, solar, heat pumps, etc.) to create Home Assistant entities.

**Key characteristics:**
- Home Assistant custom integration (not a standalone Python project)
- No `package.json` - this is pure Python
- Local polling integration (talks to heater's local API, not cloud)
- Automatic entity discovery from API metadata
- Multilingual support (entities named in DE/FR/EN from API)

## Development Commands

### Running Tests

```bash
# Install test dependencies
pip install pytest

# Run all tests
pytest tests/

# Run specific test file
pytest tests/test_discovery.py

# Run with verbose output
pytest tests/ -v

# Run with coverage (if configured)
pytest tests/ --cov=custom_components.oekofen_pellematic_compact
```

**Note:** Tests do NOT require a full Home Assistant installation. They test the discovery logic in isolation using JSON fixtures from real API responses.

### Testing in Home Assistant

Since this is a Home Assistant integration, the typical development workflow involves:

1. Copy `custom_components/oekofen_pellematic_compact/` to your Home Assistant `config/custom_components/` directory
2. Restart Home Assistant
3. Use the helper scripts if available:
   - `./start_homeassistant.sh` - Start Home Assistant for testing
   - `./monitor_logs.sh` - Monitor Home Assistant logs

## Architecture

### Core Design: Dynamic Discovery

The integration uses **dynamic entity discovery** instead of hard-coded entity definitions. This is a significant architectural decision:

**Old approach (pre-v4.0):** ~1300 lines of hard-coded sensor definitions that required manual updates for new Ökofen features.

**Current approach:** Entities are discovered dynamically from API responses at runtime.

### Key Modules

1. **`__init__.py`** - Integration setup and core hub
   - `PellematicHub`: Central data coordinator that polls the Ökofen API
   - `discover_components_from_api()`: Auto-discovers which components exist (how many heating circuits, hot water circuits, etc.)
   - `setup_platform_with_retry()`: Common setup logic for all platforms with automatic retry on failure
   - Component detection patterns: `hk1, hk2...` (heating circuits), `ww1, ww2...` (hot water), `pe1, pe2...` (pellematic heaters), etc.

2. **`dynamic_discovery.py`** - Entity discovery engine
   - `discover_all_entities()`: Main entry point - scans entire API response and creates entity definitions
   - `discover_entities_from_component()`: Processes a single component (e.g., "hk1") and determines what entities to create
   - **Entity type detection logic:**
     - `L_` prefix = read-only sensor
     - No `L_` prefix + has `min/max` = writable number entity
     - No `L_` prefix + has `format` with `|` = select entity
     - Keys like `_yesterday`, `_total`, `_runtime` = read-only statistics (even without `L_` prefix)
   - **Disambiguation:** If multiple keys in a component share the same display text, entity names get disambiguated with key suffix (e.g., "Mode (auto)" vs "Mode (eco)")
   - Infers device class, state class, units, icons from API metadata

3. **Platform files** (`sensor.py`, `number.py`, `select.py`, `climate.py`)
   - Each platform calls `discover_all_entities()` and creates Home Assistant entities from definitions
   - `sensor.py`: Read-only sensors (temperature, status, counters)
   - `number.py`: Writable numeric inputs (setpoints, limits)
   - `select.py`: Dropdown selections (modes, options)
   - `climate.py`: Climate control for heating circuits (special case, not dynamically discovered)

4. **`migration.py`** - Entity ID migration
   - Preserves old entity IDs when users upgrade from versions with different naming schemes
   - One-time migration that runs on integration setup
   - Prevents breaking automations/dashboards after upgrade

5. **`config_flow.py`** - Configuration UI
   - Handles user setup via Home Assistant's integration UI
   - Auto-discovery of components on first connection
   - Config version migration

6. **`const.py`** - Constants and helpers
   - `get_api_value()`: Extracts values from API responses (handles multiple API formats)
   - Configuration keys and defaults
   - **Important:** Hard-coded entity definitions were removed in favor of dynamic discovery (see comment at line 96)

### API Connection

The integration connects to the Ökofen heater via HTTP to fetch JSON data:

**URL Format:** `http://[IP]:[PORT]/[PASSWORD]/all?`

**Example:** `http://10.10.30.3:4321/ctT9/all?`

Where:
- `IP`: Local IP address of the heater (e.g., 10.10.30.3)
- `PORT`: TCP port for JSON interface (typically 4321)
- `PASSWORD`: Access password configured on the heater
- `all?`: Endpoint that returns all available data (note the `?` is required)

**Important:**
- The trailing `?` is required for full API response
- Timeout is 3 seconds (Ökofen recommends 2.5s minimum)
- Uses GET requests, no authentication headers needed (password is in URL)
- Writable values use format: `http://[IP]:[PORT]/[PASSWORD]/[component]_[key]=[value]`

### API Response Format

The Ökofen API returns JSON with this structure:

```json
{
  "system": {
    "L_ambient": {"val": -69, "unit": "°C", "factor": 0.1, "text": "Außentemperatur"},
    "mode": {"val": 1, "format": "0:Aus|1:Auto|2:Ein", "text": "Betriebsart"}
  },
  "hk1": {
    "L_roomtemp_act": {"val": 215, "unit": "°C", "factor": 0.1, "text": "Raumtemperatur Ist"}
  },
  "pe1": { ... },
  "ww1": { ... }
}
```

**Key properties:**
- `val`: Raw value (needs multiplication by `factor`)
- `unit`: Unit string (e.g., "°C", "kWh", "%")
- `factor`: Multiplication factor (e.g., 0.1 means divide by 10)
- `text`: Display name in heater's configured language
- `format`: For selects/enums: "0:Off|1:Auto|2:On"
- `min`/`max`: For writable numbers: valid range

### Component Naming Convention

Components follow a pattern: `{type}{number}`

- `hk1, hk2, ...` - Heating circuits (Heizkreis)
- `ww1, ww2, ...` - Hot water (Warmwasser)
- `pe1, pe2, ...` - Pellematic heaters
- `pu1, pu2, ...` - Buffer storage (Pufferspeicher)
- `sk1, sk2, ...` - Solar collectors
- `se1, se2, ...` - Solar gain sensors
- `wp1, wp2, ...` - Heat pumps (Wärmepumpe)
- `wireless1, wireless2, ...` - Wireless sensors
- `system` - Global system data (no number)
- `stirling` - Stirling engine (no number)
- `circ1` - Circulator (no number)
- `power` - Smart PV (no number)

### Data Flow

1. `PellematicHub.fetch_pellematic_data()` polls the API endpoint (default: every 30 seconds)
2. Response is parsed and stored in `hub.data`
3. Platform setup calls `discover_all_entities(hub.data)` to generate entity definitions
4. Entity classes are instantiated and registered with Home Assistant
5. Entities read their values via `get_api_value()` and update on data refresh

### Retry Mechanism

If initial entity discovery fails (API not ready, network issue), the integration automatically retries:
- Retry interval: 60 seconds
- Logs warning on first failure
- Continues retrying until successful
- User can manually trigger rediscovery via service: `oekofen_pellematic_compact.rediscover_components`

## Test Fixtures

Tests use real API responses captured from actual Ökofen systems (with passwords anonymized):

- `tests/fixtures/api_response_*.json` - Various real-world configurations
- Each fixture represents a different setup (multiple heating circuits, solar, heat pumps, etc.)
- Used for regression testing to ensure discovery works across all known configurations

## Important Patterns

### Adding Support for New API Fields

When Ökofen adds new features, the integration should automatically discover them. No code changes needed unless:

1. **New component type** (e.g., "xyz1") - Add pattern to `discover_components_from_api()` in `__init__.py`
2. **New entity type** (beyond sensor/number/select) - Add detection logic to `dynamic_discovery.py`
3. **Special handling needed** - Add specific logic to platform files

### Character Encoding

The Ökofen API uses `iso-8859-1` encoding by default. The integration:
- Allows user to configure charset in config flow
- Fixes common encoding issues (e.g., `?C` → `°C`)
- Handles control characters (newlines, tabs) in JSON strings

### API Quirks Handled

- Hotfix for Pellematic 4.02: Invalid JSON `L_statetext:` → `L_statetext":` (line 538 in `__init__.py`)
- Control character escaping in JSON strings (lines 544-553)
- URLs must end with `?` for full API response

## Home Assistant Integration Guidelines

- **No external dependencies**: `requirements: []` in manifest.json
- **Local polling**: Integration type is `local_polling`, not cloud
- **Config flow only**: No YAML configuration (removed in HA 2024+)
- **Entity registry**: All entities use unique IDs for persistence
- **Device info**: All entities grouped under single device per heater

## Version Compatibility

- **Heater firmware 3.10 or older:** Use integration v3.6.6 (last compatible version)
- **Heater firmware 4.0+:** Use integration v4.0.0+
- **Important:** Do NOT enable "compatibility mode" on the heater - it's not supported

## Common Issues to Watch For

1. **Entity ID changes:** Migration logic in `migration.py` should preserve old entity IDs, but test thoroughly when changing entity creation logic
2. **Discovery failures:** If API is slow to respond, initial setup may fail - retry mechanism handles this
3. **Duplicate entity names:** Disambiguation logic prevents this, but watch for edge cases with unusual API responses
4. **Charset issues:** If entity names show wrong characters, user may need to adjust charset setting
