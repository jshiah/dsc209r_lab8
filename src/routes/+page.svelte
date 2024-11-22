<style>@import url('$lib/global.css');</style>
<script> 
    import mapboxgl from 'mapbox-gl';
    import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
    mapboxgl.accessToken = 'pk.eyJ1IjoianNoaWFoIiwiYSI6ImNtM3Q3dXlhdzA1c3kyam92dzJibml6cTcifQ.pPf2URDQKCAxm6UaKuV7pQ';
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let stations = [];
    let map;
    
    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }
    

    onMount(async() => {
        stations = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-stations.csv');
        console.log(stations); 

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

    });
</script>

<h1>DSC209R Lab 8: Geospatial Visualization | Joyce Shiah </h1>
<p> UCSD Master's of Data Science DSC 209R Data Visualization Course | Fall '24 </p>

<div id="map">
    <svg>
        {#each stations as station}
            <circle {...getCoords(station)} r="5" fill="steelblue" />
        {/each}
    </svg>
</div>

