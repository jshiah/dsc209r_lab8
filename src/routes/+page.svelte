<style>@import url('$lib/global.css');</style>
<script> 
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    mapboxgl.accessToken = 'pk.eyJ1IjoianNoaWFoIiwiYSI6ImNtM3Q3dXlhdzA1c3kyam92dzJibml6cTcifQ.pPf2URDQKCAxm6UaKuV7pQ';
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let stations = [];
    let trips = [];
    let map;
    let mapViewChanged = 0;
    
    // Quantize scale for departure ratio
    let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);
 
    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    // Reactive scale for radius
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(stations, (d) => d.totalTraffic || 0)])
        .range([0, 25]);
    

    // Step 5: Interactive data filtering
    let timeFilter = -1; // Default to "any time"
    let formattedTime = "Any time";

    // Reactive statement to update formatted time
    $: formattedTime = timeFilter === -1
      ? "Any time"
      : new Date(0, 0, 0, Math.floor(timeFilter / 60), timeFilter % 60).toLocaleTimeString('en-US', {
          hour: 'numeric',
          minute: '2-digit',
          hour12: true
        });

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    $: filteredTrips = timeFilter === -1
        ? trips
        : trips.filter((trip) => {
            let startedMinutes = minutesSinceMidnight(trip.started_at);
            let endedMinutes = minutesSinceMidnight(trip.ended_at);
            return (
                Math.abs(startedMinutes - timeFilter) <= 60 ||
                Math.abs(endedMinutes - timeFilter) <= 60
            );
        });
    
    $: filteredArrivals = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.end_station_id
    );

    $: filteredDepartures = d3.rollup(
        filteredTrips,
        (v) => v.length,
        (d) => d.start_station_id
    );
    $: filteredStations = stations.map((station) => {
        let id = station.Number;
        station = { ...station }; // Clone the station object
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });
    
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(filteredStations, (d) => d.totalTraffic || 0)])
        .range(timeFilter === -1 ? [0, 25] : [3, 50]);






onMount(async () => {
  stations = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv');
  console.log("Stations:", stations);

  trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv').then(trips => {
    for (let trip of trips) {
      trip.started_at = new Date(trip.started_at);
      trip.ended_at = new Date(trip.ended_at);
    }
    return trips;
  });
  console.log("Processed Trips:", trips);

  // Calculate arrivals and departures
  const departures = d3.rollup(
    trips,
    (v) => v.length,
    (d) => d.start_station_id
  );
  const arrivals = d3.rollup(
    trips,
    (v) => v.length,
    (d) => d.end_station_id
  );

  // Augment stations with traffic data
  stations = stations.map((station) => {
    let id = station.Number;
    station.arrivals = arrivals.get(id) ?? 0;
    station.departures = departures.get(id) ?? 0;
    station.totalTraffic = station.arrivals + station.departures;
    return station;
  });

  console.log("Augmented Stations:", stations);



        map = new mapboxgl.Map({
            /* options */
            container: 'map', // ID of the element to contain the map
            style: 'mapbox://styles/mapbox/streets-v11', // styled monochrome: mapbox://styles/jshiah/cm3t94u84009701sq73oe0hha
            center: [-71.092761, 42.357575], // MIT HQ in Cambridge, MA: long first, then lat
            zoom: 12 // Initial zoom level
        });

        await new Promise((resolve) => map.on('load', resolve));
        map.addSource('boston_route', {
            type: 'geojson',
            data: 'https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D',
        });

        map.addLayer({
            id: 'boston_bike-network', // A name for our layer (up to you)
            type: 'line', // one of the supported layer types, e.g. line, circle, etc.
            source: 'boston_route', // The id we specified in `addSource()`
            paint: { 
                // paint params, e.g. colors, thickness, etc.
                'line-color': '#ff91a4', // Blue color for the bike routes
                'line-width': 2, // Thickness of the lines
                'line-opacity': 0.7
            }
        });

         // Step 2.4: Add the Cambridge bike lanes data source
         map.addSource('cambridge_route', {
            type: 'geojson',
            data: 'https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson'
        });

        map.addLayer({
            id: 'cambridge_bike-network',
            type: 'line',
            source: 'cambridge_route',
            paint: {
                'line-color': '#228B22',
                'line-width': 2,
                'line-opacity': 0.7
            }
        });

        map.getCanvas().addEventListener('contextmenu', (e) => {
            e.stopPropagation();
        });

        // Update mapViewChanged whenever the map is moved
        $: map?.on('move', () => mapViewChanged++);

        console.log("Filtered Trips Near Midnight:", filteredTrips);
        console.log("Filtered Stations Near Midnight:", filteredStations);
        console.log("Radius Scale Domain:", radiusScale.domain());

    });
</script>

<h1>DSC209R Lab 8: Geospatial Visualization | Joyce Shiah </h1>
<p> UCSD Master's of Data Science DSC 209R Data Visualization Course | Fall '24 </p>
<label>
    Filter by time:
    <input
      type="range"
      id="time-filter"
      min="-1"
      max="1440"
      bind:value={timeFilter}
    />
    <time>{formattedTime}</time>
    <em>{timeFilter === -1 ? "(any time)" : ""}</em>
  </label>


  <div id="map">
    <svg width="100%" height="100%">
      {#key mapViewChanged}
        {#if map}
          {#each filteredStations as station}
            <circle
              {...getCoords(station)}
              r={radiusScale(station.totalTraffic || 0)}
              style="--departure-ratio: {stationFlow(station.departures / station.totalTraffic || 0)}"
              fill-opacity="0.6"
              stroke="white"
              stroke-width="0.5"
            />
          {/each}
        {/if}
      {/key}
    </svg>
  </div>

  <div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
  </div>
  
  
  

