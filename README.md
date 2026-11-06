# LOCUS: Smarter Multi-Stop Route Planning for India

> Type in your stops. Get the shortest round trip over **real roads**, not straight lines.

LOCUS takes a list of places anywhere in India, works out how far apart they are by road, and finds the **shortest route that starts at your depot, visits every stop once, and returns**. It is built on free, open data, so there are no paid map APIs.

**Status:** backend prototype working. Frontend and quantum-optimisation stage are planned (see [Roadmap](#roadmap)).

---

## The problem

Anyone who makes several stops in one trip has to decide the order to visit them in: delivery riders, tiffin and grocery services, field technicians, college and event logistics, small local businesses. Most people do this by guesswork or by "nearest first", and both can waste a lot of distance.

The honest difficulty is that the number of possible orders grows explosively. Ten stops already give over 3 million possible orders, so picking the best one by hand (or by brute force) does not work.

## Our solution

LOCUS automates the whole chain:

1. **Find the places.** You type names such as `IISER Bhopal` or `Gateway of India, Mumbai`; the system finds them on the map, restricted to India.
2. **Measure real roads.** It asks a road-routing engine for the driving distance between every pair of places.
3. **Find the best order.** It solves the Travelling Salesman Problem **exactly**, so the route it returns is the true shortest one for the given distances.
4. **Save the data.** Everything is written to a JSON file for the next stage of the pipeline.

## How it works

```
Place names  ->  Geocoding  ->  Road distance matrix  ->  Exact TSP solver  ->  JSON output
             (Photon, India)   (OSRM, driving)          (Held-Karp)
```

| Stage | What happens | Tool |
| --- | --- | --- |
| Input | User enters `Name \| place description`; the first entry is the depot | Python `input()` |
| Geocoding | Place text becomes latitude/longitude. Results are limited to India (bounding box + country code `IN`) and the user confirms the right match | [Photon](https://photon.komoot.io) on OpenStreetMap |
| Distance matrix | One request returns driving distances between all pairs of points | [OSRM](https://project-osrm.org) Table API |
| Solver | Held-Karp dynamic programming finds the shortest round trip. It remembers the best way to reach each *(set of visited stops, current stop)* state instead of trying every order | Python (bitmask DP) |
| Output | Matrix, best route and total distance are printed and saved | `distance_matrix.json` |

### Safeguards built in

- **India-only results** so a search for "Railway Station" does not return another country.
- **Human confirmation** of each geocoded place, because place names are ambiguous.
- **Location bias toward the depot** after the first search, so ambiguous names resolve to nearby places first (a soft preference, not a restriction).
- **Road-snap warning** if a point is more than 1 km from the nearest road, which usually means the wrong place was picked.
- **Missing-route check**: the program stops with a clear error if any two points cannot be connected by road.
- **Duplicate-name check**, **request timeouts** and **polite rate limiting** towards the free public services.

## Getting started

### Requirements

- Python 3.8 or newer
- The `requests` library
- Internet access (the program calls Photon and OSRM)

### Install and run

```bash
pip install requests
python app.py
```

Before sharing the code publicly, open `app.py` and replace the contact address in the `User-Agent` header with your own. Public geocoding services ask clients to identify themselves.

### Example session

The following is an illustrative session (numbers are examples, not real measurements):

```
Enter each place as: Name | address or place description
The first location is the depot.

Depot: Depot | IISER Bhopal
Next stop (blank to finish): Station | Bhopal Junction Railway Station
Next stop (blank to finish): Airport | Raja Bhoj Airport, Bhopal
Next stop (blank to finish): 

Choose the correct match for Depot:
1. IISER Bhopal, Bhauri, Bhopal, Madhya Pradesh
Enter the matching number, or 0 to search again: 1
...

--- Shortest round trip found ---
Depot -> Airport -> Station -> Depot
Total distance: XX.XX km

Saved distance_matrix.json
```

### Output file

`distance_matrix.json` contains:

| Field | Meaning |
| --- | --- |
| `locations` | Place names, depot first |
| `coordinates_lon_lat` | Longitude and latitude of each place |
| `distances_km` | Road distance matrix in kilometres |
| `depot` | Name of the starting point |
| `shortest_route` | Best visiting order, starting and ending at the depot |
| `shortest_route_distance_km` | Total length of that route |

## Current limitations

We want to be upfront about what the prototype does not do yet:

- **Up to 18 stops.** The exact solver grows exponentially (about 2ⁿ × n² steps). It is slow in plain Python near the limit and refuses more than 18 stops.
- **Distance only.** It optimises driving distance, not travel time, traffic, tolls or delivery time windows.
- **One vehicle.** It plans a single round trip, not a fleet.
- **Public demo servers.** OSRM's public server and Photon are free community services meant for light use. A real deployment needs its own hosted instances.
- **Command line only.** There is no user interface yet.

## Roadmap

- [x] India-wide place search with user confirmation
- [x] Road distance matrix from OSRM
- [x] Exact shortest round trip (Held-Karp)
- [x] Saved JSON output for downstream use
- [ ] Build a QUBO formulation from the saved distance matrix and explore quantum / quantum-inspired optimisation
- [ ] Approximate solvers (for example nearest neighbour with 2-opt, or OR-Tools) for more than 18 stops
- [ ] Travel time as an optimisation option
- [ ] Multiple vehicles and delivery time windows
- [ ] Web or mobile frontend with a map view
- [ ] Self-hosted routing and geocoding for reliability

## Tech stack

- **Language:** Python
- **Libraries:** `requests` (plus the standard `json` and `time` modules)
- **Data and services:** OpenStreetMap, Photon, OSRM

## Project structure

```
.
├── app.py               # Input, geocoding, OSRM distance matrix, Held-Karp solver, output
├── distance_matrix.json # Generated on each run
└── README.md
```

## Data and attribution

Map data is © OpenStreetMap contributors, available under the Open Database License (ODbL). Geocoding is provided by Photon and routing by the OSRM project; please follow their usage policies.
## License

MIT License
