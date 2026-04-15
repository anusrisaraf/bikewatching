<script>
	import { onMount } from "svelte";
	import * as d3 from "d3";
	import mapboxgl from "mapbox-gl";
	import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

	const MAPBOX_ACCESS_TOKEN = import.meta.env.PUBLIC_MAPBOX_ACCESS_TOKEN;
	const STATION_DATA_URL = "https://vis-society.github.io/labs/9/data/bluebikes-stations.csv";
	const TRIP_DATA_URL = "https://vis-society.github.io/labs/9/data/bluebikes-traffic-2024-03.csv";
	const BOSTON_BIKE_LANES_URL =
		"https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D";
	const CAMBRIDGE_BIKE_LANES_URL =
		"https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson";

	mapboxgl.accessToken = MAPBOX_ACCESS_TOKEN || "";

	let map;
	let stations = [];
	let trips = [];
	let mapViewChanged = 0;

	let arrivals = new Map();
	let departures = new Map();

	let timeFilter = -1;
	let selectedStation = null;
	let isochrone = null;
	let mapError = "";

	let radiusScale = d3.scaleSqrt().domain([0, 0]).range([0, 25]);
	const stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);

	const departuresByMinute = Array.from({ length: 1440 }, () => []);
	const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

	const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
	const profile = "cycling";
	const minutes = [5, 10, 15, 20];
	const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];

	$: timeFilterLabel = new Date(0, 0, 0, 0, Math.max(timeFilter, 0)).toLocaleString("en", {
		timeStyle: "short"
	});

	$: filteredDepartures =
		timeFilter === -1
			? departures
			: d3.rollup(
					filterByMinute(departuresByMinute, timeFilter),
					(v) => v.length,
					(d) => d.start_station_id
				);

	$: filteredArrivals =
		timeFilter === -1
			? arrivals
			: d3.rollup(
					filterByMinute(arrivalsByMinute, timeFilter),
					(v) => v.length,
					(d) => d.end_station_id
				);

	$: filteredStations = stations.map((station) => {
		const id = station.Number;
		const arr = filteredArrivals.get(id) ?? 0;
		const dep = filteredDepartures.get(id) ?? 0;
		return {
			...station,
			arrivals: arr,
			departures: dep,
			totalTraffic: arr + dep
		};
	});

	$: radiusScale = d3
		.scaleSqrt()
		.domain([
			0,
			d3.max(stations, (d) => (arrivals.get(d.Number) ?? 0) + (departures.get(d.Number) ?? 0)) || 0
		])
		.range([3, 30]);

	$: if (selectedStation) {
		getIso(+selectedStation.Long, +selectedStation.Lat);
	} else {
		isochrone = null;
	}

	onMount(async () => {
		if (!MAPBOX_ACCESS_TOKEN) {
			mapError = "Missing Mapbox token. Set PUBLIC_MAPBOX_ACCESS_TOKEN for local dev or GitHub secret for deploy.";
			console.error("Missing PUBLIC_MAPBOX_ACCESS_TOKEN");
			return;
		}
		await Promise.all([initializeMap(), loadStationAndTripData()]);
	});

	async function initializeMap() {
		map = new mapboxgl.Map({
			container: "map",
			style: "mapbox://styles/mapbox/streets-v12",
			center: [-71.09415, 42.36027],
			zoom: 12,
			minZoom: 10,
			maxZoom: 17
		});

		map.on("move", () => {
			mapViewChanged += 1;
		});

		await new Promise((resolve) => map.on("load", resolve));

		map.addSource("boston_route", {
			type: "geojson",
			data: BOSTON_BIKE_LANES_URL
		});

		map.addSource("cambridge_route", {
			type: "geojson",
			data: CAMBRIDGE_BIKE_LANES_URL
		});

		const bikeLanePaint = {
			"line-color": "rgb(250, 0, 25)",
			"line-width": 3,
			"line-opacity": 0.4
		};

		map.addLayer({
			id: "boston-bike-lanes",
			type: "line",
			source: "boston_route",
			paint: bikeLanePaint
		});

		map.addLayer({
			id: "cambridge-bike-lanes",
			type: "line",
			source: "cambridge_route",
			paint: bikeLanePaint
		});
	}

	async function loadStationAndTripData() {
		stations = await d3.csv(STATION_DATA_URL);

		trips = await d3.csv(TRIP_DATA_URL).then((rows) => {
			for (const trip of rows) {
				trip.started_at = new Date(trip.started_at);
				trip.ended_at = new Date(trip.ended_at);

				const startedMinutes = minutesSinceMidnight(trip.started_at);
				const endedMinutes = minutesSinceMidnight(trip.ended_at);
				departuresByMinute[startedMinutes].push(trip);
				arrivalsByMinute[endedMinutes].push(trip);
			}
			return rows;
		});

		departures = d3.rollup(trips, (v) => v.length, (d) => d.start_station_id);
		arrivals = d3.rollup(trips, (v) => v.length, (d) => d.end_station_id);
	}

	function getCoords(station) {
		if (!map) return { cx: -1000, cy: -1000 };
		const point = new mapboxgl.LngLat(+station.Long, +station.Lat);
		const { x, y } = map.project(point);
		return { cx: x, cy: y };
	}

	function minutesSinceMidnight(date) {
		return date.getHours() * 60 + date.getMinutes();
	}

	function filterByMinute(tripsByMinute, minute) {
		let minMinute = (minute - 60 + 1440) % 1440;
		let maxMinute = (minute + 60) % 1440;

		if (minMinute > maxMinute) {
			let beforeMidnight = tripsByMinute.slice(minMinute);
			let afterMidnight = tripsByMinute.slice(0, maxMinute);
			return beforeMidnight.concat(afterMidnight).flat();
		}
		return tripsByMinute.slice(minMinute, maxMinute).flat();
	}

	async function getIso(lon, lat) {
		const base = `${urlBase}${profile}/${lon},${lat}`;
		const params = new URLSearchParams({
			contours_minutes: minutes.join(","),
			contours_colors: contourColors.join(","),
			polygons: "true",
			access_token: mapboxgl.accessToken
		});
		const url = `${base}?${params.toString()}`;
		const query = await fetch(url, { method: "GET" });
		isochrone = await query.json();
	}

	function geoJSONPolygonToPath(feature) {
		const path = d3.path();
		const rings = feature.geometry.coordinates;

		for (const ring of rings) {
			for (let i = 0; i < ring.length; i += 1) {
				const [lng, lat] = ring[i];
				const { x, y } = map.project([lng, lat]);
				if (i === 0) path.moveTo(x, y);
				else path.lineTo(x, y);
			}
			path.closePath();
		}
		return path.toString();
	}
</script>

<svelte:head>
	<title>BikeWatch</title>
	<meta name="description" content="Interactive bike traffic map of Boston and Cambridge" />
</svelte:head>

<header>
	<h1>🚲 BikeWatch</h1>
	<label>
		Filter by time:
		<input type="range" min="-1" max="1440" bind:value={timeFilter} />
		{#if timeFilter !== -1}
			<time>{timeFilterLabel}</time>
		{:else}
			<em>(any time)</em>
		{/if}
	</label>
</header>

<p>
	Interactive bike map of Boston/Cambridge with bike lanes, station demand, traffic flow filtering, and
	station isochrones.
</p>

{#if mapError}
	<p class="error">{mapError}</p>
{/if}

<div id="map">
	<svg>
		{#key mapViewChanged}
			{#if isochrone}
				{#each isochrone.features as feature}
					<path
						d={geoJSONPolygonToPath(feature)}
						fill={feature.properties.fillColor}
						fill-opacity="0.2"
						stroke="#000000"
						stroke-opacity="0.5"
						stroke-width="1"
					>
						<title>{feature.properties.contour} minutes of biking</title>
					</path>
				{/each}
			{/if}
			{#each filteredStations as station}
				<circle
					{...getCoords(station)}
					r={radiusScale(station.totalTraffic)}
					class={station?.Number === selectedStation?.Number ? "selected" : ""}
					role="button"
					tabindex="0"
					aria-label={`Toggle isochrone for ${station.NAME}`}
					style="--departure-ratio: {stationFlow((station.departures || 0) / (station.totalTraffic || 1))}"
					on:mousedown={() =>
						(selectedStation = selectedStation?.Number !== station?.Number ? station : null)}
				>
					<title>
						{station.totalTraffic} trips ({station.departures} departures, {station.arrivals} arrivals)
					</title>
				</circle>
			{/each}
		{/key}
	</svg>
</div>

<div class="legend">
	<div style="--departure-ratio: 1">More departures</div>
	<div style="--departure-ratio: 0.5">Balanced</div>
	<div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
	@import url("$lib/global.css");

	header {
		display: flex;
		gap: 1em;
		align-items: baseline;
	}

	header label {
		margin-left: auto;
	}

	header time,
	header em {
		display: block;
	}

	header em {
		color: #666;
		font-style: italic;
	}

	#map {
		position: relative;
		flex: 1;
		min-height: 70vh;
		background: #f2f2f2;
	}

	:global(.mapboxgl-canvas) {
		cursor: default;
	}

	#map svg {
		position: absolute;
		z-index: 1;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}

	#map svg circle,
	.legend > div {
		--color-departures: steelblue;
		--color-arrivals: darkorange;
		--color: color-mix(
			in oklch,
			var(--color-departures) calc(100% * var(--departure-ratio)),
			var(--color-arrivals)
		);
	}

	#map svg circle {
		fill: var(--color);
		fill-opacity: 0.6;
		stroke: white;
		stroke-width: 1;
		pointer-events: auto;
		transition: opacity 0.2s ease;
	}

	#map svg:has(circle.selected) circle:not(.selected) {
		opacity: 0.3;
	}

	#map svg path {
		pointer-events: auto;
	}

	.legend {
		display: flex;
		gap: 1px;
		margin-block: 1rem 0;
	}

	.legend > div {
		flex: 1;
		background: var(--color);
		padding: 0.5em 1.2em;
		color: white;
		font-size: 0.875rem;
	}

	.legend > div:nth-child(1) {
		text-align: left;
	}

	.legend > div:nth-child(2) {
		text-align: center;
	}

	.legend > div:nth-child(3) {
		text-align: right;
	}

	.error {
		color: #b00020;
		font-weight: 600;
	}
</style>
