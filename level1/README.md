# Access Log Aggregator

## Overview

`data_processor.py` is a command-line processor for web request logs. It groups requests by UTC hour and endpoint, then writes request counts, nearest-rank latency percentiles (p50, p95, p99), the number of server errors (HTTP 5xx), and the error rate in parts per million. It is designed to process large files in chunks and can use multiple processes on supported systems.

This module does **not** define a `DataProcessor` class and does not implement general numerical dataset statistics, filtering, transformations, or CSV persistence. The documentation below describes the checked-in source file's actual interface and behavior.

## Requirements & Setup

- Python 3.7 or later.
- Standard-library modules: `gc`, `marshal`, `os`, `sys`, `array`, `bisect`, `itertools`; the fallback also imports `datetime`, and the rank histogram helper imports `collections`.
- The optimized path uses POSIX APIs including `os.fork` and `os.pread`; run it on a POSIX system such as Linux. The reference fallback is also selected for input that does not match fast-path assumptions, but it does not make the whole program portable to platforms without the POSIX APIs used by `fast_main`.

Save the script as `data_processor.py`. By default it reads `/data/requests.log` and writes `/out/summary.txt`. Override those directories with command-line options:

```sh
python3 data_processor.py --data ./data --out ./out
```

The `--data` directory must contain `requests.log`. The output directory is created if needed. You can also use `--data=PATH` and `--out=PATH`. Run `python3 data_processor.py --help` to print the usage line.

To import the module and call its driver:

```python
import data_processor

data_processor.main("./data", "./out")
```

## Input and Output Format

Each input record is one space-separated line with five fields:

```text
timestamp method endpoint status latency_us
```

For example:

```text
2025-02-14T10:15:30.123Z GET /api/items 200 840
```

The reference parser expects timestamps accepted by `datetime.strptime` with `%Y-%m-%dT%H:%M:%S.%f%z`, including a fractional second and timezone. The optimized parser assumes timestamp strings ending in `Z` and uses their first 13 bytes as the hour key. Status and latency must be integer-convertible; latency values must fit an unsigned 32-bit integer for the optimized path.

The output is sorted by (hour, endpoint), with one record per group:

```text
YYYY-MM-DDTHH:00:00Z endpoint count p50 p95 p99 errors errors_per_million
```

Percentiles use the nearest-rank rule: rank `ceil(p * count / 100)`, with ranks starting at one. The error rate is integer floor division: `errors * 1_000_000 // count`.

## API Reference

The public callable interface consists of the functions below. The checked-in script does not expose the requested `DataProcessor` methods such as `calculate_mean()`, `filter_data()`, or CSV import/export.

### `main(data_dir, out_dir)`

Runs the processor using the supplied input and output directories.

- `data_dir` (`str` or path accepted by `os.path.join`): directory containing `requests.log`.
- `out_dir` (`str` or path accepted by `os.path.join`): destination directory for `summary.txt`.
- Returns: `None`.
- Errors: The function catches processing exceptions and retries with the single-process optimized path, then the reference parser. Errors from the final fallback, including file I/O and malformed records, can propagate.

### `fast_main(data_dir, out_dir, single=False)`

Runs the chunked optimized implementation.

- `data_dir`: input directory.
- `out_dir`: output directory.
- `single` (`bool`, default `False`): disable process parallelism when true.
- Returns: `None`.
- Errors: Can raise `OSError` for file and process operations, `Fallback` when input violates fast-path assumptions, or `WorkerFailed` when a child process fails.

### `reference_main(data_dir, out_dir)`

Runs the straightforward Python reference algorithm.

- `data_dir`: input directory.
- `out_dir`: output directory.
- Returns: `None`.
- Errors: File I/O exceptions can propagate. Malformed lines can raise `ValueError` (for example, wrong field count or invalid integers), and invalid timestamps can raise `ValueError` from `datetime.strptime`.

### `cpu_budget()`

Returns the detected worker count, cached after its first calculation. The `A_INGEST_WORKERS` environment variable overrides detection; otherwise CPU availability and, where available, a cgroup quota are considered.

- Parameters: none.
- Returns: positive `int`.
- Errors: Invalid `A_INGEST_WORKERS` values can raise `ValueError`.

### Internal helpers and exceptions

Names beginning with an underscore (`_parse_args`, `_percentile`, `_count_errors`, and others) are implementation details and are not stable public APIs. `Fallback` and `WorkerFailed` are internal exception classes used by the optimized path.

## Examples

### Process a log from the command line

Create `data/requests.log`:

```text
2025-02-14T10:15:30.123Z GET /api/items 200 840
2025-02-14T10:16:01.500Z POST /api/items 503 2200
2025-02-14T11:02:00.000Z GET /health 200 40
```

Then run:

```sh
python3 data_processor.py --data ./data --out ./out
cat ./out/summary.txt
```

The output contains two grouped rows: one for `/api/items` in the 10:00 hour (two requests, one 5xx error), and one for `/health` in the 11:00 hour.

### Call the driver from Python

```python
from data_processor import main

main(data_dir="./data", out_dir="./out")
```

The program writes the result to `./out/summary.txt`; it does not return statistics as Python objects.

## Error Handling & Edge Cases

- **Empty input file:** produces an empty `summary.txt`.
- **Missing input file or inaccessible paths:** filesystem errors may ultimately propagate from `main` after its retries.
- **Malformed records:** the fast path detects several mismatches and falls back to the reference algorithm. The reference parser expects exactly five fields separated by single spaces and valid timestamp, status, and latency values; malformed data can raise `ValueError`.
- **Fast-path constraints:** data with unusual timestamp layout or values not representable as unsigned 32-bit latency values causes a fallback. The fallback parses integer latencies without that 32-bit representation restriction.
- **Process failures:** `main` retries without multiprocessing before using the reference implementation.
- **No numerical-array API:** there is no empty-dataset mean/median behavior, numeric type validation API, data filtering or transformation API, or CSV persistence API in this source file.
- **Output replacement:** each successful run opens `summary.txt` for writing, replacing its prior contents.

## Environment Variables

- `A_INGEST_CHUNK`: chunk size in bytes for fast-path parsing; defaults to 256 KiB.
- `A_INGEST_WORKERS`: optional positive worker count override. Invalid integer text raises `ValueError` during worker detection.
