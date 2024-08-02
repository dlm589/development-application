<script>
    import { onMount, afterUpdate } from "svelte";
    import maplibregl from "maplibre-gl";
    import development from "../data/development-application.geo.json";
    import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
    import positron from "../data/positron.json";
    //import BaseLayer from "../data/pmtiles.json";
    import * as pmtiles from "pmtiles";
    /*
    var wards = [ward1,ward2,ward3,ward4,ward5,ward6,ward7,ward8,ward9,ward10,ward11,ward12,ward13,ward14,ward15,ward16,ward17,ward18,ward19,
    ward20,ward21,ward22,ward23,ward24,ward25]

    //import geojsonvt from "geojson-vt";
    //import property from "../data/properties.geo.json";
    //import developments from "../data/development-application.geojson"
    */
    let map;
    let address = [];
    let info = [];
    let height = [];
    let application = [];
    let description = [];
    let application_url = [];
    let submission_date = [];
    let popupContent = false;
    let query = ""; //This is the input address from users.
    let distance = ""; // This is the input for buffer distance
    let dist;
    let lat;
    let lon;
    let results;

    // Application Type Filter
    let allFilter = true;
    let ozFilter = false;
    let spaFilter = false;
    let sbFilter = false;
    let plFilter = false;
    let cdFilter = false;
    let mvFilter = false;
    let coFilter = false;

    //Height Filter
    let highFilter = false;
    let midFilter = false;
    let lowFilter = false;
    let noHFilter = false;

    //Date Filter
    let oneMonthFilter = false;
    let twoMonthFilter = false;
    let threeMonthFilter = false;
    let sixMonthFilter = false;
    let yearFilter = false;

    let applicationfilter = [];
    let datefilter = [];
    let heightfilter = [];

    function applicationFilter() {
        applicationfilter = [];
        map.setFilter("development-select", [
            "==",
            ["get", "APPLICATION#"],
            "",
        ]);

        // when true, set filter
        if (ozFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "OZ"]);
        }
        if (spaFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "SA"]);
        }
        if (sbFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "SB"]);
        }
        if (plFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "PL"]);
        }
        if (cdFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "CD"]);
        }
        if (mvFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "MV"]);
        }
        if (coFilter) {
            applicationfilter.push(["==", ["get", "APPLICATION_TYPE"], "CO"]);
        }

        /* Sorting the filters with other filters, this ensures that filters from the same
        filter function are sorted together */

        if ((datefilter.length > 0) & (heightfilter.length == 0)) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...datefilter],
            ];
            //console.log("Date", filter)
        } else if (heightfilter.length > 0 && datefilter.length == 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...heightfilter],
            ];
            //console.log("Height", filter)
        } else if (heightfilter.length > 0 && datefilter.length > 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...heightfilter],
                ["any", ...datefilter],
            ];
            //console.log("Height, Date", filter)
        } else {
            var filter = ["any", ...applicationfilter];
            //console.log("Application", filter)
        }

        /* When people unclick buttons, sometimes all the points can disappear,
         to solve this problem, we can remove filters with nothing but "any" in there */
        for (let i = 0; i < filter.length; i++) {
            // if the one of the filters in the filter list only has "any" left, then
            // length is 1, so remove it.
            if (filter[i].length == 1) {
                filter.splice(filter.indexOf(filter[i]), 1);
            }
        }
        if (filter.length == 2) {
            //then there
            filter[0] = "any";
            map.setFilter("development-ID", filter);
        } else if (filter.length == 1) {
            // This is to remove all filters when there is just one "any" in the filter
            map.setFilter("development-ID", null);
            allFilter = true;
        } else {
            map.setFilter("development-ID", filter);
        }
    }

    /* filtering height */

    function heightFilter() {
        map.setFilter("development-select", [
            "==",
            ["get", "APPLICATION#"],
            "",
        ]);
        heightfilter = [];

        if (highFilter)
            heightfilter.push([
                "==",
                ["get", "HEIGHT"],
                "High-rise (15+ Storeys)",
            ]);
        if (midFilter)
            heightfilter.push([
                "==",
                ["get", "HEIGHT"],
                "Mid-rise (5-14 Storeys)",
            ]);
        if (lowFilter)
            heightfilter.push([
                "==",
                ["get", "HEIGHT"],
                "Low-rise (0-4 Storeys)",
            ]);
        if (noHFilter)
            heightfilter.push(["==", ["get", "HEIGHT"], "No Height Info"]);

        if (applicationfilter.length > 0 && datefilter.length == 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...heightfilter],
            ];
            //console.log("Application", filter);
        } else if ((datefilter.length > 0) & (applicationfilter.length == 0)) {
            var filter = [
                "all",
                ["any", ...heightfilter],
                ["any", ...datefilter],
            ];
            //console.log("Date", filter);
        } else if (datefilter.length > 0 && applicationfilter.length > 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...heightfilter],
                ["any", ...datefilter],
            ];
            //console.log("Date & Application", filter);
        } else {
            var filter = ["any", ...heightfilter];
            //console.log("Height", filter);
        }

        /* When people unclick buttons, sometimes all the points can disappear,
         to solve this problem, we can remove filters with nothing but "any" in there */
        for (let i = 0; i < filter.length; i++) {
            // if the one of the filters in the filter list only has "any" left, then
            // length is 1, so remove it.
            if (filter[i].length == 1) {
                filter.splice(filter.indexOf(filter[i]), 1);
            }
        }
        if (filter.length == 2) {
            //then there

            filter[0] = "any";

            map.setFilter("development-ID", filter);
        } else if (filter.length == 1) {
            // This is to remove all filters when there is just one "any" in the filter
            map.setFilter("development-ID", null);
            allFilter = true;
        } else {
            map.setFilter("development-ID", filter);
        }
    }

    /* filtering date */
    function dateFilter() {
        map.setFilter("development-select", [
            "==",
            ["get", "APPLICATION#"],
            "",
        ]);

        datefilter = [];

        if (oneMonthFilter) {
            datefilter.push(["<=", ["get", "DATE_DISTANCE"], 30.0]);
        }
        if (twoMonthFilter) {
            datefilter.push(["<=", ["get", "DATE_DISTANCE"], 60.0]);
        }
        if (threeMonthFilter) {
            datefilter.push(["<=", ["get", "DATE_DISTANCE"], 90.0]);
        }
        if (sixMonthFilter) {
            datefilter.push(["<=", ["get", "DATE_DISTANCE"], 180.0]);
        }
        if (yearFilter) {
            datefilter.push(["<=", ["get", "DATE_DISTANCE"], 365.0]);
        }

        if (applicationfilter.length > 0 && heightfilter == 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...datefilter],
            ];
            console.log("Application", filter);
        } else if (heightfilter.length > 0 && applicationfilter.length == 0) {
            var filter = [
                "all",
                ["any", ...heightfilter],
                ["any", ...datefilter],
            ];
            console.log("Height", filter);
        } else if (heightfilter.length > 0 && applicationfilter.length > 0) {
            var filter = [
                "all",
                ["any", ...applicationfilter],
                ["any", ...heightfilter],
                ["any", ...datefilter],
            ];
            console.log("Height & Application", filter);
        } else {
            var filter = ["any", ...datefilter];
            console.log("Date", filter);
        }

        /* When people unclick buttons, sometimes all the points can disappear,
         to solve this problem, we can remove filters with nothing but "any" in there */
        for (let i = 0; i < filter.length; i++) {
            // if the one of the filters in the filter list only has "any" left, then
            // length is 1, so remove it.
            if (filter[i].length == 1) {
                filter.splice(filter.indexOf(filter[i]), 1);
            }
        }
        if (filter.length == 2) {
            filter[0] = "any";
            map.setFilter("development-ID", filter);
        } else if (filter.length == 1) {
            // This is to remove all filters when there is just one "any" in the filter
            map.setFilter("development-ID", null);
            allFilter = true;
        } else {
            map.setFilter("development-ID", filter);
        }
    }

    onMount(() => {
        let protocol = new pmtiles.Protocol();

        maplibregl.addProtocol("pmtiles", protocol.tile);
        map = new maplibregl.Map({
            container: "map",
            style: positron, //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
            center: [-79.9, 43.68], // starting position
            minZoom: 9,
            maxZoom: 19,
            projection: "globe",
            scrollZoom: true,
            attributionControl: false,
        });

        //let protoLayers = BaseLayer;
        // Adding zoom and rotation controls to the map
        // Adding additional layers from geojson
        map.on("load", function () {
            // Add the source with the concatenated attribution string
            for (let i = 1; i <= 25; i++) {
                let num;
                if (i < 10) {
                    num = `0${i}`;
                } else {
                    num = `${i}`
                }

                map.addSource(`ward${num}`, {
                    type: "vector",
                    url:
                        "pmtiles://" +
                        `/development-application/Ward${i}.pmtiles`,
                });

                map.addLayer({
                    "id": `ward${num}_layer`,
                    "type": "line",
                    "source": `ward${num}`,
                    "source-layer": `property_ward_${num}`,
                    "layout": {},
                    "paint": {
                        "line-color": "#888888", // Border color
                        "line-width": 0.2, // Border width
                    },
                });

            }

            map.addSource("development", {
                type: "geojson",
                data: development,
            });
            map.addLayer({
                id: "development-ID",
                type: "circle",
                source: "development",
                paint: {
                    "circle-color": [
                        "match",
                        ["get", "APPLICATION_TYPE"],
                        "OZ",
                        "#1e3765",
                        "SA",
                        "#2a6f97",
                        "MV",
                        "#DAA520",
                        "#a9d6e5",
                    ],
                    "circle-radius": 5,
                },
            });
            map.addLayer({
                id: "development-select",
                type: "circle",
                source: "development",
                filter: ["==", ["get", "APPLICATION#"], ""],
                paint: {
                    "circle-color": '#000000',
                    "circle-opacity": 0,
                    "circle-radius": 5,
                    "circle-stroke-color": "red",
                    "circle-stroke-width": 2,
                },
            });

            map.on("click", "ward13_layer", function (e) {
                console.log(
                    e.features[0].properties.Addresses,
                    e.features[0].properties.Aream2,
                );
            });

        });

        // Boundary of map
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
            // clear the lists first, so old values don't interfere
            address = [];
            info = [];
            height = [];
            application = [];
            description = [];
            application_url = [];
            submission_date = [];

            // Process the overlapping points
            overlappingPoints.forEach((point) => {
                address.push(point.properties.STREET);
                info.push(point.properties.INFO);
                height.push(point.properties.HEIGHT);
                application.push(point.properties["APPLICATION#"]);
                description.push(point.properties.DESCRIPTION);
                application_url.push(point.properties.APPLICATION_URL);
                submission_date.push(point.properties.DATE_SUBMITTED);
            });
            if (application[0] != null){
                map.setFilter("development-select", [
                "==",
                ["get", "APPLICATION#"],
                application[0],
            ]);

            } else {
                map.setFilter("development-select", [
                "==",
                ["get", "STREET"],
                address[0],
            ]);
            }
            
            console.log(application)
            //$: title = e.features[0].properties.CSDNAME;

            popupContent = true;
        });
    });
    // Geocoder for people to input their address and zoom to input address
    const baseUrl =
        "https://nominatim.openstreetmap.org/search.php?format=jsonv2&q=";

    const getResults = async () => {
        results = await fetch(baseUrl + query + ", Toronto").then((res) =>
            res.json(),
        );
        if (results.length > 0) {
            //this is to remove the previous address point searched (if true)
            if (map.getSource(`address ${lon}`)) {
                
                map.removeLayer(`address-layer ${lon}`);
                map.removeSource(`address ${lon}`);
                
                map.removeLayer(`buffer-layer ${lon} ${dist}`);
                map.removeSource(`buffer ${lon} ${dist}`);
            }
            if (distance == "") {
                distance = 500;
            }
            dist = distance;
            //get long - lat
            lat = +results[0].lat;
            lon = +results[0].lon;

            /* CREATE BUFFER */
            var point = turf.point([lon, lat]);
            var buffered = turf.buffer(point, distance, { units: "meters" });

            // add point to show the searched address
            map.addSource(`buffer ${lon} ${dist}`, {
                type: "geojson",
                data: buffered,
            });
            
            map.addLayer({
                id: `buffer-layer ${lon} ${dist}`,
                type: "line",
                source: `buffer ${lon} ${dist}`,
                paint: {
                    "line-color": "red",
                    "line-width": 5, // Set the color of the point
                },
            });

            // Calculate the bounding box of the polygon
            var bbox = turf.bbox(buffered);

            /* CREATE BUFFER */
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

            // Get the coordinates of the buffer
            var coordinates = buffered.geometry.coordinates;

            // Extract all coordinates of the buffer
            var allCoordinates = [];
            coordinates.forEach(function (ring) {
                ring.forEach(function (coord) {
                    allCoordinates.push(coord);
                });
            });

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
</script>

<main>
    <div id="map" class="map" />

    <div class="info-panel">
        <h1>Application Information Centre</h1>

        <p>Last Updated: July 31, 2024</p>

        <!-- THIS BUTTON ALLOWS PEOPLE TO SELECT DEVELOPMENT APPLICATION TYPES-->
        <div class="buttons-box">
            <h3>Filter By Application Type</h3>
            <button
                class="application-button"
                on:click={() => {
                    allFilter = !allFilter;
                    console.log(allFilter)
                    if (allFilter == true){
                        map.setFilter("development-ID", null);
                    } else {
                        map.setFilter("development-ID", ["==",
                        ["get", "APPLICATION#"],
                        "",]);
                        map.setFilter("development-select", ["==",
                        ["get", "APPLICATION#"],
                        "",]);
                    }
                    
                    applicationfilter = [];
                    datefilter = [];
                    heightfilter = [];
                    ozFilter = false;
                    spaFilter = false;
                    cdFilter = false;
                    sbFilter = false;
                    plFilter = false;
                    mvFilter = false;
                    coFilter = false;
                    highFilter = false;
                    midFilter = false;
                    lowFilter = false;
                    noHFilter = false;
                    oneMonthFilter = false;
                    twoMonthFilter = false;
                    threeMonthFilter = false;
                    sixMonthFilter = false;
                    yearFilter = false;
                }}
                style="background-color: {allFilter
                    ? '#1e3765'
                    : ''}; color: {allFilter ? 'white' : 'black'}">ALL</button
            >
            <button
                class="application-button"
                on:click={() => {
                    ozFilter = !ozFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {ozFilter
                    ? '#1e3765'
                    : ''}; color: {ozFilter ? 'white' : 'black'}"
                >OPA & ZBA</button
            ><button
                class="application-button"
                on:click={() => {
                    spaFilter = !spaFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {spaFilter
                    ? '#2a6f97'
                    : ''}; color: {spaFilter ? 'white' : 'black'}"
                >SITE PLAN</button
            ><button
                class="application-button"
                on:click={() => {
                    cdFilter = !cdFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {cdFilter
                    ? '#a9d6e5'
                    : ''}; color: {cdFilter ? 'black' : 'black'}">DRAFT CONDO</button
            ><button
                class="application-button"
                on:click={() => {
                    sbFilter = !sbFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {sbFilter
                    ? '#a9d6e5'
                    : ''}; color: {sbFilter ? 'black' : 'black'}">SUBDIVISION</button
            ><button
                class="application-button"
                on:click={() => {
                    plFilter = !plFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {plFilter ? '#a9d6e5' : ''}; color: {plFilter ? 'black' : 'black'}">PL</button
            > <button
            class="application-button"
            on:click={() => {
                mvFilter = !mvFilter;
                allFilter = false;
                applicationFilter();
            }}
            style="background-color: {mvFilter ? '#DAA520' : ''}; color: {mvFilter ? 'white' : 'black'}">MV</button
                > <button
                class="application-button"
                on:click={() => {
                    coFilter = !coFilter;
                    allFilter = false;
                    applicationFilter();
                }}
                style="background-color: {coFilter ? '#a9d6e5' : ''}; color: {coFilter ? 'black' : 'black'}">CO</button
            > <br />
            <h3>Filter By Building Height</h3>
            <button
                class="application-button"
                on:click={() => {
                    highFilter = !highFilter;
                    allFilter = false;
                    heightFilter();
                }}
                style="background-color: {highFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">High-rise (15+ Storeys)</button
            ><button
                class="application-button"
                on:click={() => {
                    midFilter = !midFilter;
                    allFilter = false;
                    heightFilter();
                }}
                style="background-color: {midFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">Mid-rise (5-14 Storeys)</button
            ><button
                class="application-button"
                on:click={() => {
                    lowFilter = !lowFilter;
                    allFilter = false;
                    heightFilter();
                }}
                style="background-color: {lowFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">Low-rise (0-4 Storeys)</button
            ><button
                class="application-button"
                on:click={() => {
                    noHFilter = !noHFilter;
                    allFilter = false;
                    heightFilter();
                }}
                style="background-color: {noHFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">No Height Info</button
            >

            <!--SEARCH BY DATE-->
            <h3>Filter By Date</h3>
            <button
                class="application-button"
                on:click={() => {
                    oneMonthFilter = !oneMonthFilter;
                    twoMonthFilter = false;
                    threeMonthFilter = false;
                    sixMonthFilter = false;
                    yearFilter = false;
                    allFilter = false;
                    dateFilter();
                }}
                style="background-color: {oneMonthFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">1 Month</button
            ><button
                class="application-button"
                on:click={() => {
                    twoMonthFilter = !twoMonthFilter;
                    oneMonthFilter = false;
                    threeMonthFilter = false;
                    sixMonthFilter = false;
                    yearFilter = false;
                    allFilter = false;
                    dateFilter();
                }}
                style="background-color: {twoMonthFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">2 Months</button
            ><button
                class="application-button"
                on:click={() => {
                    threeMonthFilter = !threeMonthFilter;
                    oneMonthFilter = false;
                    twoMonthFilter = false;
                    sixMonthFilter = false;
                    yearFilter = false;
                    allFilter = false;
                    dateFilter();
                }}
                style="background-color: {threeMonthFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">3 Months</button
            ><button
                class="application-button"
                on:click={() => {
                    sixMonthFilter = !sixMonthFilter;
                    oneMonthFilter = false;
                    twoMonthFilter = false;
                    threeMonthFilter = false;
                    yearFilter = false;
                    allFilter = false;
                    dateFilter();
                }}
                style="background-color: {sixMonthFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">6 Months</button
            ><button
                class="application-button"
                on:click={() => {
                    yearFilter = !yearFilter;
                    oneMonthFilter = false;
                    twoMonthFilter = false;
                    threeMonthFilter = false;
                    sixMonthFilter = false;
                    allFilter = false;
                    dateFilter();
                }}
                style="background-color: {yearFilter
                    ? '#a9d6e5'
                    : ''}; color: 'black'">12 Months</button
            >
        </div>

        <h3><b>Search Address</b></h3>
        <input bind:value={query} placeholder="i.e. 100 St George St" />
        <input
            bind:value={distance}
            placeholder="i.e. 500, distance is in meters"
        /> <br />
        <button
            class="address-button"
            on:click={getResults}
            disabled={query.length < 1}>Search</button
        >
        {#if popupContent}
            <h2>Development Applications</h2>
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
                    <p>
                        {submission_date[i]} <br />
                        {info[i]} <br />
                        {height[i]}
                    </p>
                    <p>{description[i]}</p>
                    <p></p>
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
        padding-top: 0px;
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
        padding-top: 0px;
        padding-left: 10px;
        padding-right: 20px;
        padding-bottom: 5px;
        margin-bottom: 20px;
    }
    a {
        color: #41729f;
    }
    .application-button {
        font-size: 12px;
        width: auto;
        height: 28px;
        left: 10px;
        color: black;
        margin-right: 5px;
        margin-bottom: 5px;
        border-width: 0px;
        position: relative;
        font-weight: bold;
    }

    /* Searh Address */
    input {
        left: 10px;
        width: 24vw;
        height: 20px;

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
        left: 10px;
        width: 24%;
        height: 28px;

        border-width: 0px;
        margin-right: 5px;

        font-weight: bold;
        position: relative;
        border-color: grey;
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
