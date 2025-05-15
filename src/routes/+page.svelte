<script>
    import * as d3 from "d3";
    import mapboxgl from "mapbox-gl";
    import "mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";

    let map;
    let stations = [];
    let mapViewChanged = 0;
    let trips = [];
    let departures, arrivals;
    let radiusScale;
    let timeFilter = -1;
    let departuresByMinute = Array.from({ length: 1440 }, () => []);
    let arrivalsByMinute = Array.from({ length: 1440 }, () => []);
    let filteredDepartures,
        filteredArrivals,
        filteredStations = [];

    mapboxgl.accessToken =
        "pk.eyJ1Ijoic2Nyb2NoYSIsImEiOiJjbWFwaDJzYXAwMGQ5MmpwdjA3dnllZXd6In0.IO_oWUSLgcJszXWeZEHUrQ";

    async function initMap() {
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/streets-v12",
            zoom: 12,
            center: [-71.0788727679991, 42.36199480654666],
        });

        await new Promise((resolve) => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addLayer({
            id: "boston-bike-lanes",
            type: "line",
            source: "boston_route",
            layout: { "line-join": "round", "line-cap": "round" },
            paint: {
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.7,
            },
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "cambridge-bike-lanes",
            type: "line",
            source: "cambridge_route",
            layout: { "line-join": "round", "line-cap": "round" },
            paint: {
                "line-color": "red",
                "line-width": 3,
                "line-opacity": 0.7,
            },
        });

        map.on("move", () => mapViewChanged++);
    }

    async function loadStationData() {
        try {
            const csvUrl =
                "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv";
            const data = await d3.csv(csvUrl);
            stations = data.map((s) => ({
                id: s.Number,
                name: s.NAME,
                Lat: +s.Lat,
                Long: +s.Long,
            }));
            console.log("Stations loaded:", stations.length);
        } catch (error) {
            console.error("Error loading station data:", error);
        }
    }

    async function loadStationDemand() {
        try {
            const csvUrl =
                "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv";
            const data = await d3.csv(csvUrl);
            trips = data.map((trip) => ({
                id: trip.ride_id,
                started_at: new Date(trip.started_at),
                ended_at: new Date(trip.ended_at),
                start_station_id: trip.start_station_id,
                end_station_id: trip.end_station_id,
            }));
            // popula buckets por minuto para desempenho
            for (const trip of trips) {
                const sm = minutesSinceMidnight(trip.started_at);
                departuresByMinute[sm].push(trip);
                const em = minutesSinceMidnight(trip.ended_at);
                arrivalsByMinute[em].push(trip);
            }
            // console.log('Trips loaded:', trips.length);
        } catch (error) {
            console.error("Error loading trip data:", error);
        }
    }

    onMount(async () => {
        initMap();
        await loadStationData();
        await loadStationDemand();
    });

    $: if (stations.length && trips.length) {
        departures = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.start_station_id,
        );
        arrivals = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.end_station_id,
        );
        stations = stations.map((station) => {
            const id = station.id;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    }

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
        timeStyle: "short",
    });

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function filterByMinute(tripsByMinute, minute) {
        const minMinute = (minute - 60 + 1440) % 1440;
        const maxMinute = (minute + 60) % 1440;
        if (minMinute > maxMinute) {
            const before = tripsByMinute.slice(minMinute);
            const after = tripsByMinute.slice(0, maxMinute);
            return before.concat(after).flat();
        }
        return tripsByMinute.slice(minMinute, maxMinute).flat();
    }

    $: filteredDepartures =
        timeFilter === -1
            ? d3.rollup(
                  trips,
                  (v) => v.length,
                  (d) => d.start_station_id,
              )
            : d3.rollup(
                  filterByMinute(departuresByMinute, timeFilter),
                  (v) => v.length,
                  (d) => d.start_station_id,
              );
    $: filteredArrivals =
        timeFilter === -1
            ? d3.rollup(
                  trips,
                  (v) => v.length,
                  (d) => d.end_station_id,
              )
            : d3.rollup(
                  filterByMinute(arrivalsByMinute, timeFilter),
                  (v) => v.length,
                  (d) => d.end_station_id,
              );

    $: filteredStations = stations.map((station) => {
        const id = station.id;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return {
            ...station,
            arrivals: arr,
            departures: dep,
            totalTraffic: arr + dep,
        };
    });

    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(filteredStations, (d) => d.totalTraffic) || 0])
        .range(timeFilter === -1 ? [0, 25] : [3, 30]);

    function getCoords(station) {
        const point = new mapboxgl.LngLat(station.Long, station.Lat);
        const { x, y } = map.project(point);
        return { cx: x, cy: y };
    }
</script>

<header>
    <h1>🚲 BikeWatch</h1>
    <label for="timeFilterSlider">Filter by time:</label>
    <input
        id="timeFilterSlider"
        type="range"
        min="-1"
        max="1440"
        bind:value={timeFilter}
    />
    {#if timeFilter !== -1}
        <time>{timeFilterLabel}</time>
    {:else}
        <em>(any time)</em>
    {/if}
</header>

<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each filteredStations as station}
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                />
            {/each}
        {/key}
    </svg>
</div>

<style>
    @import url("$lib/global.css");

    :global(header) {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }
    :global(header label) {
        margin-left: auto;
    }
    :global(header time),
    :global(header em) {
        display: block;
    }
    :global(header em) {
        color: #666;
        font-style: italic;
    }

    #map {
        flex: 1;
        background-color: #eee;
        position: relative;
    }

    #map svg {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        z-index: 1;
        pointer-events: none;
    }

    #map svg circle {
        fill-opacity: 0.6;
        stroke: white;
    }
</style>
