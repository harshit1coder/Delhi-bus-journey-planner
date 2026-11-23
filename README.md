# Delhi Bus Journey Planner

A bus journey planner for Delhi that predicts travel times from **observed** bus GPS data instead of the published timetable.

Built for the **Transport Stack Hackathon 2026, Problem 1**. The whole project is one Google Colab notebook: [`delhi_bus_journey_planner.ipynb`](delhi_bus_journey_planner.ipynb).

## Results

| Test | Our model | Timetable / reference |
|---|---|---|
| In-vehicle time, held-out day (14 Oct), 30,306 stop pairs: median abs. % error | **11.7%** | 23.9% (timetable) |
| In-vehicle time: mean abs. error | **5.3 min** | 8.6 min (timetable) |
| In-vehicle time: pairs within 20% of observed | **71%** | 43% (timetable) |
| End-to-end journeys vs Google Maps (12 comparisons): MAE / median abs. % error | 17.6 min / 21.2% | Google Maps bus-only time |

- The timetable is weakest at night (00-05: median error 48%) and in the evening peak (17-20: bias -27%).
- The model's own error is largest in the evening peak (median error 14.6%, bias -8.9%).
- The end-to-end planner is clearly less accurate than the in-vehicle model (see [Limitations](#limitations)).

## How it works

```
GPS archive (SQLite/day) ──► clean ──► map-match to GTFS stops ──► stop-to-stop travel times
                                                                         │
GTFS static schedule ───────────────────────────────────────────────────┤
                                                                         ▼
                                              median travel time model (stop pair × day type × hour)
                                                                         │
                                                                         ▼
                                   time-dependent Dijkstra planner ──► FastAPI + Leaflet map page
```

1. **Data.** DTC bus GPS pings for October 2026 (GTFS-realtime archived as one SQLite file per day) plus the static GTFS schedule. Days used: 7, 8 and 11 Oct for training, 14 Oct held out for validation.
2. **Linking GPS to GTFS.** GPS trip IDs never match GTFS IDs exactly. 3-part IDs (`route_hh_mm`) match the schedule 95.6% of the time; 4-part IDs (with a suffix) only 6.5%. Only 3-part trips found in GTFS are used (37,449 of 82,125 distinct trips on 8 Oct, about 46%).
3. **Cleaning.** Delhi bounding box, duplicate `(trip, timestamp)` removal, and GPS spike removal (a ping is dropped if the jumps both from the previous and to the next ping imply more than 28 m/s).
4. **Map-matching.** Each trip's pings are matched to its scheduled stops (a ping within 120 m counts as "at the stop"). Consecutive hit stops give one observed travel time. Segments are dropped if travel time is outside 0-20 min, stops are under 20 m apart, or straight-line speed exceeds 25 m/s.
5. **Model.** Median observed travel time per stop pair at three levels: (stop pair, day type, hour) with at least 3 observations, then (stop pair, day type) with at least 2, then (stop pair). Unseen pairs fall back to distance divided by the citywide median speed for that hour. Fitted on training days only.
6. **Planner.** Time-dependent Dijkstra on arrival time over route-stop nodes, with four moves:
   - **walk** to a stop within 350 m (distance × 1.3 detour factor, 4.5 km/h), after getting off a bus
   - **board** a route, paying an expected wait (half the average headway from scheduled trips, scaled by schedule density at that hour, limited to 2-20 min)
   - **ride** to the next stop using the learned travel time at the current time of day
   - **alight**

   Origin and destination snap to up to 6 stops within 800 m. Two options are returned: the fastest, and one that penalises each transfer by 15 minutes. Each option also reports the same journey planned at 23:00 as a late-night reference.
7. **Serving.** FastAPI REST service with a click-on-the-map Leaflet page.

## Data

| Input | Source |
|---|---|
| GPS archive (about 15 GB compressed, 31 daily SQLite files) | `https://pub-091bd9e794bb4877b10b29e6df80f065.r2.dev/2026-10/delhi-dtc-bus-gtfs-rt-2026-10.7z` |
| Static GTFS schedule | `delhi_buses_static_gtfs_v1.zip` (not part of the archive; upload when the notebook asks) |

The data is not included in this repository.

## Running it

The notebook is designed for **Google Colab** (it uses `/content`, `google.colab.files`, and a Cloudflare tunnel).

**Requirements**
- About 30 GB of free disk (15 GB archive plus about 13.4 GB of extracted daily databases)
- The static GTFS zip, uploaded when prompted in Step 1
- Python packages installed by the first cell: `fastapi`, `uvicorn`, `nest_asyncio`, plus `aria2` and `p7zip-full` via apt. The notebook also uses `numpy`, `pandas`, `scipy`, `matplotlib` and `requests` (preinstalled on Colab)

**Steps**
1. Open the notebook in Colab.
2. Run the cells top to bottom. Cleaning and segmentation skip days already processed, so re-running after a disconnect is cheaper.
3. Upload `delhi_buses_static_gtfs_v1.zip` when asked.
4. Section 8 starts the API on port 8000 and prints a public `trycloudflare.com` URL.

The public link is temporary: it works only while the Colab runtime is alive, and a new URL is issued each time the tunnel cell is run.

## API

| Endpoint | Purpose |
|---|---|
| `GET /journeys?from=&to=&depart=HH:MM&day=mon..sun` | Up to two journey options. `from` and `to` are `lat,lng` or a stop name. `depart` is 24-hour IST. |
| `GET /stops?q=&limit=` | Search stops by name |
| `GET /health` | Service status |
| `GET /` | Map page: click an origin, then a destination |
| `GET /docs` | Interactive API documentation |

**Example**

```
GET /journeys?from=28.6675,77.2282&to=28.5491,77.2512&depart=08:30&day=tue
```

Kashmere Gate ISBT to Nehru Place, Tuesday 08:30. The response contains `options`, each with `total_min`, `transfers`, `arrive`, a list of `legs` (walk and bus legs with board/alight stops, stops count, wait and ride minutes) and the 23:00 reference time.

Errors return JSON with an `error` field: 400 for malformed input or coordinates outside Delhi, 404 when no journey is found.

## Outputs

Section 8 writes these files to `/content/outputs` and zips them as `final_outputs.zip`:

| File | Content |
|---|---|
| `validation_trips.csv` | Per-pair observed, model and timetable minutes for the held-out day |
| `validation_scatter.png` | Predicted vs observed in-vehicle time, model and timetable |
| `google_comparison.csv` | Planner output for 10 origin-destination pairs × 3 departures (09:00, 14:00, 22:30), with Google Maps values where recorded |
| `sample_0.json` to `sample_2.json` | Example API responses |

## Evaluation method

**In-vehicle time (held-out day).** For each cleaned trip on 14 Oct, up to two random stop pairs 8 to 35 stops apart are drawn from stops the bus actually passed. The observed time from GPS is compared with (a) the sum of learned segment times along the way, advancing the clock as it goes, and (b) the scheduled time difference. Metrics: MAE, median absolute error, median absolute percentage error, median signed bias, and share within 20%.

**End-to-end journeys.** Ten origin-destination pairs are planned at 09:00, 14:00 and 22:30 on a Tuesday. Bus-only Google Maps times were recorded by hand for 12 of the 30 combinations and compared with the planner's fastest option (which includes walking and waiting).

## Limitations

- **Little training data.** 4 of 31 days were used: two weekdays and one Sunday to fit, one weekday to validate. The weekend model rests on a single day.
- **Only linkable trips.** About half of the GPS trips can be tied to scheduled stops; the rest are ignored. A bus is credited with only about 51-58% of its scheduled stops because a ping must be within 120 m of the stop.
- **Waiting time is a schedule-based estimate**, not an observed headway. It depends on scheduled trip counts per route and hour and an assumed 18-hour service day, and is the weakest part of the end-to-end estimate.
- **Validation covers in-vehicle time only**, not walking, waiting or transfers. It also uses the plain lookup (hourly, then day-type, then any-hour median, then distance) without the hour-of-day speed rescaling the planner applies, so it is a close but not identical test of the planner's ride times.
- **The Google Maps comparison is small and manual** (12 values), and the date and time basis of those values is not recorded.
- **The Google Maps comparison is not an independent test.** The waiting-time scaling was tuned and the comparison re-run on the same values, so it can look better than it would on fresh journeys.
- **Map lines are straight** between boarding and alighting stops; the GTFS `shapes.txt` road geometry is not used.
- **Public link is temporary** and disappears when the Colab runtime stops.

## Possible next steps

- Train on more of the 31 available days, especially more weekend days.
- Estimate waiting time from observed headways in the GPS data.
- Recover more GPS trips (the 4-part IDs) to raise stop coverage.
- Draw bus legs along `shapes.txt` instead of straight lines.
- Validate end-to-end journeys against a larger, independent, timestamped set of reference times.

## Repository layout

```
.
├── delhi_bus_journey_planner.ipynb   # full pipeline, API and map page
├── README.md
├── LICENSE                           # MIT
└── outputs/                          # validation CSVs, scatter plot, sample JSON
```

## Acknowledgements

- DTC bus GPS archive and static GTFS schedule provided for the Transport Stack Hackathon 2026
- Map tiles © OpenStreetMap contributors; map page built with Leaflet
