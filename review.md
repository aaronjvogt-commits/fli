# Code Review: `fli` Repository

Review of [`punitarani/fli`](https://github.com/punitarani/fli) — a Python library providing programmatic access to Google Flights data via reverse-engineered API.

---

## Summary

The codebase is well-structured with good separation of concerns (core/search/models/cli/mcp layers), strong Pydantic validation throughout, and thoughtful reverse-engineering of the Google Flights API payload format. The main areas for improvement are: a mutation side effect in date chunking, deprecated Pydantic usage, a non-thread-safe singleton, duplicated formatting code, and a few serialization inconsistencies in the MCP layer.

---

## Critical / High Priority

### 1. `SearchDates.search` mutates the caller's `filters` object (`fli/search/dates.py`)

In the date-range chunking loop, `segment.travel_date` is overwritten in-place on the passed-in `filters` object:

```python
for segment in filters.flight_segments:
    segment.travel_date = (
        datetime.strptime(segment.travel_date, "%Y-%m-%d")
        + timedelta(days=self.MAX_DAYS_PER_SEARCH)
    ).strftime("%Y-%m-%d")
```

This is a silent, destructive side effect. The caller's filters are permanently altered after a chunked search, making it impossible to safely call `search()` twice with the same filters object. Fix by copying the segment or the whole filters before mutation, e.g. using `deepcopy(filters)` the same way `SearchFlights.search` does, or by computing the chunk dates independently without touching `filters`.

### 2. Deprecated Pydantic `.dict()` method (`fli/models/google_flights/flights.py`, `fli/models/google_flights/dates.py`)

Both `FlightSearchFilters.format()` and `DateSearchFilters.format()` call:

```python
serialize(obj.dict(exclude_none=True))
```

Pydantic v2 deprecated `.dict()` in favour of `.model_dump()`. This produces `PydanticDeprecatedSince20` warnings at runtime and will break in a future Pydantic release. Replace with `.model_dump(exclude_none=True)`.

### 3. Non-thread-safe global singleton (`fli/search/client.py`)

```python
client = None

def get_client() -> Client:
    global client
    if not client:
        client = Client()
    return client
```

In the HTTP MCP server (`fli-mcp-http`) or any multi-threaded caller, two threads can simultaneously see `client is None` and each create a `Client` instance. While the Python GIL prevents full data corruption, you can still end up with two sessions (and two connection pools) open. Fix with a module-level lock:

```python
import threading
_client_lock = threading.Lock()
_client: Client | None = None

def get_client() -> Client:
    global _client
    if _client is None:
        with _client_lock:
            if _client is None:
                _client = Client()
    return _client
```

### 4. Pydantic v2 validator mutation anti-pattern (`fli/models/google_flights/base.py`, `fli/models/google_flights/dates.py`)

`TimeRestrictions.validate_latest_times` and `DateSearchFilters.validate_date_order` attempt to swap field values by writing into `info.data`:

```python
info.data[field_prefix] = v   # TimeRestrictions
info.data["from_date"] = v    # DateSearchFilters
```

In Pydantic v2 `ValidationInfo.data` is a read-only snapshot of previously-validated fields. Writes to it are silently dropped—the intended swap never happens. The swap logic is dead code. Replace with a `@model_validator(mode='before')` that reorders the fields before individual field validators run, or raise a `ValueError` requiring the caller to supply values in the correct order.

---

## Medium Priority

### 5. Duplicated `format()` / `serialize()` logic (`fli/models/google_flights/flights.py`, `fli/models/google_flights/dates.py`)

`FlightSearchFilters.format()` and `DateSearchFilters.format()` share ~80 % of their code, including the entire `serialize()` inner function defined twice, the segment-formatting block, and the bag/layover/airlines/emissions blocks. Extract `serialize()` to a module-level utility in `fli/models/google_flights/` and factor the shared segment-formatting logic into a helper function to reduce the ~150 lines of duplicated code and the associated maintenance burden.

### 6. MCP `_serialize_flight_leg` returns raw enum objects (`fli/mcp/server.py`)

```python
def _serialize_flight_leg(leg: Any) -> dict[str, Any]:
    return {
        "departure_airport": leg.departure_airport,  # Airport enum — not JSON-serializable
        "arrival_airport":   leg.arrival_airport,    # Airport enum
        "airline":           leg.airline,            # Airline enum
        "airline_code":      getattr(leg.airline, "name", leg.airline).lstrip("_"),
        ...
    }
```

The CLI's `serialize_flight_leg` (in `fli/cli/utils.py`) correctly converts these to dicts with `code`/`name` keys. The MCP version relies on FastMCP to handle enum serialisation, which may produce inconsistent output (e.g., `<Airport.JFK: 'John F. Kennedy International Airport'>` in some contexts). Use the same approach as the CLI: return string codes/names explicitly.

### 7. `search_dates` MCP tool missing `emissions` filter (`fli/mcp/server.py`)

`_execute_date_search` constructs a `DateSearchFilters` without an `emissions` parameter even though `DateSearchFilters` exposes one. The `DateSearchParams` model also has no `emissions` field. Users cannot filter low-emissions dates via MCP. Add the parameter to parity with `search_flights`.

### 8. Confusing exception chain in `Client.get/post` (`fli/search/client.py`)

The retry/rate-limit decorator stack is:

```python
@sleep_and_retry
@limits(calls=10, period=1)
@retry(stop=stop_after_attempt(3), wait=wait_exponential(), reraise=True)
def post(self, url, **kwargs):
    try:
        ...
    except Exception as e:
        raise Exception(f"POST request failed: {str(e)}") from e
```

Two problems:
- The blanket `except Exception` wraps every error (including HTTP 4xx responses that should not be retried) in a generic `Exception`, losing the original type for callers.
- Tenacity sits *inside* the rate limiter, so all three retry attempts count against the 10 req/sec budget simultaneously. A burst of three fast failures can consume three rate-limit slots in rapid succession.

Fix: let the `raise_for_status()` exception propagate naturally (remove the try/except), and consider moving `@retry` outside `@limits` if retries should not count as new requests.

### 9. `serialize_flight_result` 2-leg pricing inconsistency (`fli/mcp/server.py`)

```python
# Multi-city (3+ legs) or 2-leg non-round-trip: combined price on the
# final leg (matches Google Flights pricing and the CLI display logic).
price_segment = segments[-1] if len(segments) > 2 else segments[0]
```

The comment says "combined price on the final leg" but for exactly 2 segments (non-round-trip) the code takes `segments[0]` (first leg), not the last. The CLI `display_flight_results` does the same (`segments[-1] if num_legs > 2 else segments[0]`), so behaviour is consistent between CLI and MCP — but the comment is misleading and the logic may be wrong for 2-leg multi-city trips where the final price lives on the last leg. Clarify the comment and verify the pricing assumption against real API responses.

---

## Low Priority / Style

### 10. `httpx` listed as a direct dependency but never imported (`pyproject.toml`)

`httpx>=0.28.1` is in the main `[dependencies]` block but no production code in `fli/` imports it (only `curl_cffi` is used). If it is pulled in transitively by another dependency, remove it as a direct dependency to avoid confusion; if it is used somewhere, verify the import exists.

### 11. `plotext` should be optional (`pyproject.toml`)

`plotext` is used only in `fli/cli/utils.py` for sparkline charts. It is listed as a required runtime dependency, meaning every user — including those who only use the MCP server or the Python API — installs a plotting library they will never use. Move it to a `cli` optional-dependency group.

### 12. Package name vs. module name mismatch (`pyproject.toml`)

The package is published as `flights` on PyPI (`name = "flights"`) but the importable module is `fli`. `pip install fli` will fail with "package not found"; users must know to run `pip install flights`. This is a common source of confusion. Consider adding `fli` as a PyPI alias or making the discrepancy more prominent in the README installation section.

### 13. Time range does not validate `start <= end` (`fli/core/parsers.py`, `fli/cli/utils.py`)

Both `parse_time_range` and `validate_time_range` accept any two values in `[0, 23]` without checking that `start_hour <= end_hour`. A range like `"20-6"` is silently accepted. If overnight ranges are not intended (the Google Flights UI doesn't support them), add: `if start_hour > end_hour: raise ParseError(...)`.

### 14. `run_http` bypasses `FlightSearchConfig` for host/port (`fli/mcp/server.py`)

```python
def run_http(host: str = "127.0.0.1", port: int = 8000) -> None:
    env_host = os.getenv("HOST")
    env_port = os.getenv("PORT")
```

This uses raw `os.getenv` while the rest of the server uses `pydantic_settings`. For consistency, add `host` and `port` fields to `FlightSearchConfig` (with env-prefix `FLI_MCP_HOST` / `FLI_MCP_PORT`) and read them from `CONFIG`.

### 15. `FlightSegment.validate_travel_date` rejects past dates, complicating tests (`fli/models/google_flights/base.py`)

The validator raises `ValueError` for any past travel date. This means unit tests must always use future dates or patch `datetime.now()`. Any test fixtures using historical flight data require explicit mocking. Consider whether this validation belongs in the model layer or the application layer (e.g., the CLI/MCP entry points), and document the test strategy in `CLAUDE.md`.

### 16. `MAX_PAST_FROM_DATE_DAYS = 6` undocumented magic constant (`fli/models/google_flights/dates.py`)

No comment explains why 6 days is the grace period for `from_date` being in the past. Add a brief explanation (e.g., "Google Flights calendar still shows prices for dates up to N days ago").

### 17. `python-dotenv` redundant direct dependency (`pyproject.toml`)

`python-dotenv` is already pulled in by `pydantic-settings` (an optional dependency). If the project relies on it being available, it should move to the `mcp` extras group (where `pydantic-settings` already lives) rather than being a required top-level dependency.

---

## Positive Observations

- **Thorough reverse-engineering notes**: The inline comments in `FlightSearchFilters.format()` documenting which API payload indices have known vs. unknown effects are invaluable for maintainability.
- **Shared core utilities**: Extracting parsers and builders into `fli/core/` so both CLI and MCP share the same parsing logic is a clean architectural decision that prevents divergence.
- **Rate limiting + retries**: The dual-layer protection (ratelimit + tenacity) is appropriate for an unofficial API client.
- **Pydantic throughout**: Comprehensive use of Pydantic models with field validators ensures data integrity before hitting the API.
- **Protobuf currency decoder**: The hand-rolled varint parser in `fli/core/currency.py` to extract ISO currency codes from opaque Google tokens is a clever and well-structured piece of reverse engineering.
- **Test structure**: The `tests/` tree mirrors the source tree, and the `--fuzz` / `--all` marker gating keeps CI fast while preserving extended coverage options.
