<script>
    import { onMount, afterUpdate } from "svelte";
    import maplibregl from "maplibre-gl";
    import development from "../data/development-application.geo.json";
    import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities

    //import property_boundary from "../data/property-boundary.geo.json"
    //import  from "../data/transit-lines.geo.json";
    //import notVancouverPolygon from '../../data/not-vancouver-polygon.geo.json';

    let map;
    let address = [];
    let info = [];
    let height = [];
    let application = [];
    let description = [];
    let application_url = [];
    let popupContent = false;
    let query = ""; //This is the input address from users.
    let lat;
    let lon;
    let results;

    // Filter
    let ozFilter = false;
    let spaFilter = false;
    let sbFilter = false;
    let plFilter = false;
    let cdFilter = false;

    function applyFilter() {
        const filter = [];

        if (ozFilter) filter.push(["==", ["get", "APPLICATION_TYPE"], "OZ"]);
        if (spaFilter) filter.push(["==", ["get", "APPLICATION_TYPE"], "SA"]);
        if (sbFilter) filter.push(["==", ["get", "APPLICATION_TYPE"], "SB"]);
        if (plFilter) filter.push(["==", ["get", "APPLICATION_TYPE"], "PL"]);
        if (cdFilter) filter.push(["==", ["get", "APPLICATION_TYPE"], "CD"]);

        map.setFilter("development-ID", ["any", ...filter]);
        console.log(ozFilter);
        console.log(spaFilter);
        console.log(sbFilter);
        console.log(plFilter);
        console.log(cdFilter);
    }

    onMount(() => {
        map = new maplibregl.Map({
            container: "map",
            style: "https://basemaps.cartocdn.com/gl/positron-gl-style/style.json", //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
            center: [-79.9, 43.68], // starting position
            minZoom: 2,
            maxZoom: 19,
            projection: "globe",
            scrollZoom: true,
            attributionControl: false,
        });

        // Adding zoom and rotation controls to the map
        // Adding additional layers from geojson
        map.on("load", function () {
            const layers = map.getStyle().layers;

            map.addSource("development", {
                type: "geojson",
                data: development,
            });

            map.addLayer({
                id: "development-ID",
                type: "circle",
                source: "development",
                paint: {
                    "circle-color": "#1e3765",
                    "circle-radius": 5,
                },
                //before: "transit-line-bold-ID",
            });
            map.addLayer({
                id: "development-select",
                type: "circle",
                source: "development",
                filter: ["==", ["get", "APPLICATION#"], ""],
                paint: {
                    "circle-color": "#0047AB",
                    "circle-radius": 5,
                    "circle-stroke-color": "red",
                    "circle-stroke-width": 2,
                },
                //before: "transit-line-bold-ID",
            }); /*
            map.addSource("parcel", {
                type: "geojson",
                data: property_boundary,
            });
            map.addLayer({
                id: "parcel-lines",
                type: "line",
                source: "parcel",
                layout: {},
                paint: {
                    "line-color": "#DC4633", // Border color
                    "line-width": 4, // Border width
                },
                //before: "transit-line-bold-ID",
            });
            // map libre is not really smart, you will have to create a symbol layer to label your circle layer*/
        });

        map.fitBounds([
            [-79.14904366238247, 43.87527014932047],
            [-79.60668327438583, 43.56196116510192],
        ]);
        map.on("mouseenter", "development-ID", () => {
            map.getCanvas().style.cursor = "pointer";
        });
        map.on("mouseleave", "development-ID", () => {
            map.getCanvas().style.cursor = "";
        });
        map.on("click", "development-ID", (e) => {
            const tolerance = 0.0; // Adjust the tolerance as needed
            const clickedPoint = e.point;

            // Find all points within the tolerance radius of the clicked point
            const overlappingPoints = map.queryRenderedFeatures(
                [
                    [clickedPoint.x - tolerance, clickedPoint.y - tolerance],
                    [clickedPoint.x + tolerance, clickedPoint.y + tolerance],
                ],

                { layers: ["development-ID"] },
            );
            console.log(clickedPoint);
            address = [];
            info = [];
            height = [];
            application = [];
            description = [];
            application_url = [];
            // Process the overlapping points
            overlappingPoints.forEach((point) => {
                //console.log(point);

                address.push(point.properties.STREET);
                info.push(point.properties.INFO);
                height.push(point.properties.HEIGHT);
                application.push(point.properties["APPLICATION#"]);
                description.push(point.properties.DESCRIPTION);
                application_url.push(point.properties.APPLICATION_URL);
            });

            map.setFilter("development-select", [
                "==",
                ["get", "APPLICATION#"],
                application[0],
            ]);

            //$: title = e.features[0].properties.CSDNAME;

            popupContent = true;
        });
    });
    // Geocoder for people to input their address and zoom to input address
    const baseUrl =
        "https://nominatim.openstreetmap.org/search.php?format=jsonv2&q=";

    const getResults = async () => {
        results = await fetch(baseUrl + query + ", Toronto").then((res) => res.json());
        if (results.length > 0) {
            //this is to remove the previous address point searched (if true)
            if (map.getSource(`address ${lon}`)) {
                map.removeSource(`address ${lon}`);
                map.removeLayer(`address-layer ${lon}`);
                map.removeSource(`buffer ${lon}`);
                map.removeLayer(`buffer-layer ${lon}`);
            }

            //get long - lat
            lat = +results[0].lat;
            lon = +results[0].lon;
            console.log(lat, lon); /*
            map.flyTo({
                // These options control the ending camera position: centered at
                // the target, at zoom level 16, and north up.
                center: [lon, lat],
                zoom: 16,
                bearing: 0,
                // These options control the flight curve, making it move slowly and zoom out almost completely before starting
                // to pan.
                speed: 2, // make the flying slow
                curve: 1, // change the speed at which it zooms out
                // This can be any easing function: it takes a number between 0 and 1 and returns another number between 0 and 1.
                easing(t) {
                    return t;
                },
                // this animation is considered essential with respect to prefers-reduced-motion
                essential: true,
            });*/

            /* CREATE BUFFER */
            var point = turf.point([lon, lat]);
            var buffered = turf.buffer(point, 500, { units: "meters" });

            // add point to show the searched address
            map.addSource(`buffer ${lon}`, {
                type: "geojson",
                data: buffered,
            });
            map.addLayer({
                id: `buffer-layer ${lon}`,
                type: "line",
                source: `buffer ${lon}`,
                paint: {
                    "line-color": "red",
                    "line-width": 5, // Set the color of the point
                },
            });
            map.addSource(`address ${lon}`, {
                type: "geojson",
                data: {
                    type: "Point",
                    coordinates: [lon, lat],
                },
            });
            map.addLayer({
                id: `address-layer ${lon}`,
                type: "circle",
                source: `address ${lon}`,
                paint: {
                    "circle-radius": 8,
                    "circle-color": "#FF0000", // Set the color of the point
                },
            });

            // Get the coordinates of the polygon
            var coordinates = buffered.geometry.coordinates;

            // Extract all coordinates of the polygon
            var allCoordinates = [];
            coordinates.forEach(function (ring) {
                ring.forEach(function (coord) {
                    allCoordinates.push(coord);
                });
            });

            // Calculate the bounding box of the polygon
            var bbox = turf.bbox(buffered);

            // Fit the map to the bounding box
            map.fitBounds(
                [
                    [bbox[0], bbox[1]],
                    [bbox[2], bbox[3]],
                ],
                { padding: 20 },
            );
        } else {
            alert("Sorry, no geocoding results for " + query);
        }
    };

    console.log(ozFilter);
</script>

<main>
    <div id="map" class="map" />

    <div class="info-panel">
        <h1>Development Applications in Toronto</h1>
        <p>
            Map Created By <a
                href="https://www.linkedin.com/in/chun-fu-liu/"
                target="_blank"
                >Michael Liu
            </a>
        </p>

        <!-- THIS BUTTON ALLOWS PEOPLE TO SELECT DEVELOPMENT APPLICATION TYPES-->
        <div class="buttons-box">
            <button
                class="application-button"
                on:click={() => {
                    ozFilter = !ozFilter;
                    applyFilter();
                }}
                style="background-color: {ozFilter
                    ? '#1e3765'
                    : ''}; color: {ozFilter ? 'white' : 'black'}"
                >OPA & ZBA</button
            ><button
                class="application-button"
                on:click={() => {
                    spaFilter = !spaFilter;
                    applyFilter();
                }}
                style="background-color: {spaFilter
                    ? '#1e3765'
                    : ''}; color: {spaFilter ? 'white' : 'black'}"
                >SITE PLAN</button
            ><button
                class="application-button"
                on:click={() => {
                    cdFilter = !cdFilter;
                    applyFilter();
                }}
                style="background-color: {cdFilter
                    ? '#1e3765'
                    : ''}; color: {cdFilter ? 'white' : 'black'}"
                >DRAFT CONDO</button
            ><button
                class="application-button"
                on:click={() => {
                    sbFilter = !sbFilter;
                    applyFilter();
                }}
                style="background-color: {sbFilter
                    ? '#1e3765'
                    : ''}; color: {sbFilter ? 'white' : 'black'}"
                >SUBDIVISION</button
            ><button
                class="application-button"
                on:click={() => {
                    plFilter = !plFilter;
                    applyFilter();
                }}
                style="background-color: {plFilter
                    ? '#1e3765'
                    : ''}; color: {plFilter ? 'white' : 'black'}">PL</button
            >

            <!--SEARCH ADDRESS-->
        </div>
        <h3><b>Search Address</b></h3>
        <input
            bind:value={query}
            placeholder="i.e. 100 St George St, Toronto"
        />
        <button
            class="address-button"
            on:click={getResults}
            disabled={query.length < 1}>Search</button
        >

        {#if popupContent}
            {#each info as inf, i}
                <div class="application">
                    <h2>{address[i]}</h2>
                    {#if application_url[i] === undefined}
                        <h3>{application[i]}</h3>
                    {:else if application_url[i] != undefined}
                        <h3>
                            <a href={application_url[i]} target="_blank"
                                >{application[i]}</a
                            >
                        </h3>
                    {/if}
                    <p>{info[i]}</p>
                    <p>{height[i]}</p>

                    <p>{description[i]}</p>
                </div>
            {/each}
        {/if}
    </div>
</main>

<style>
    main {
        overflow-x: hidden;
    }
    .map {
        height: 100%;
        width: 65%;
        top: 0;
        left: 35%;
        position: absolute;
        overflow: hidden;
    }
    .info-panel {
        position: absolute;
        top: 0px;
        left: 0px;
        width: 35vw;
        height: 100%;
        font-size: 17px;
        font-family: sans-serif;
        background-color: rgb(254, 251, 249, 1);
        color: #1e3765;
        overflow-x: hidden;
        scrollbar-width: 1px;
    }
    .application {
        left: 10px;
        width: 35vw - 20px;
        height: auto;
        padding-top: 2px;
        background-color: #d3d3d3;
        margin-right: 20px;
        margin-top: 2px;
        position: relative;
        scrollbar-width: 1px;
    }
    .buttons-box {
        left: 0px;
        width: 35vw - 20px;
        height: auto;
        padding-top: 2px;
        background-color: white;
        margin-right: 20px;
        position: relative;
    }
    h1 {
        font-size: 30px;
        padding-top: 20px;
        padding-left: 10px;
        /*padding-bottom: 3px;*/
        margin: 0px;
        margin-bottom: 0px;
        color: #1e3765;
        /* background-color: #F1C500; */
        /* -webkit-text-stroke: 1px #6FC7EA; */
        font-weight: bold;
    }
    h2 {
        font-size: 20px;

        padding-top: 20px;
        padding-left: 10px;

        /*padding-bottom: 3px;*/
        margin: 0px;
        margin-bottom: 0px;
        color: #1e3765;
        /* background-color: #F1C500; */
        /* -webkit-text-stroke: 1px #6FC7EA; */
        font-weight: bold;
    }
    h3 {
        font-size: 18px;

        padding-top: 10px;
        padding-left: 10px;

        /*padding-bottom: 3px;*/
        margin: 0px;
        margin-bottom: 0px;
        color: #41729f;
        /* background-color: #F1C500; */
        /* -webkit-text-stroke: 1px #6FC7EA; */
        font-weight: bold;
    }
    p {
        padding-left: 10px;
        padding-right: 20px;
        padding-bottom: 10px;
    }
    a {
        color: #41729f;
    }
    .application-button {
        font-size: 12px;
        width: auto;
        height: 28px;
        left: 10px;
        margin-right: 5px;
        margin-bottom: 5px;
        border-width: 0px;
        position: relative;
        font-weight: bold;
    }

    /* Searh Address */
    input {
        width: 24vw;
        height: 20px;
        left: 10px;
        color: #6d247a;
        border-width: 1px;
        margin-right: 5px;

        padding-left: 2px;
        padding-top: 3px;
        padding-bottom: 3px;
        font-weight: bold;
        position: relative;
        border-color: grey;
    }
    .address-button {
        font-size: 12px;
        width: auto;
        height: 28px;
        left: 0px;

        margin-bottom: 20px;
        border-width: 1px;
        position: relative;
    }
    /* SCROLL BARS */
    ::-webkit-scrollbar {
        width: 5px;
    } /* Track */
    ::-webkit-scrollbar-track {
        box-shadow: inset 0 0 5px grey;
        border-radius: 5px;
    }

    /* Handle */
    ::-webkit-scrollbar-thumb {
        background: #41729f;
        border-radius: 5px;
    }

    /* Handle on hover */
    ::-webkit-scrollbar-thumb:hover {
        background: #41729f;
    }
</style>
