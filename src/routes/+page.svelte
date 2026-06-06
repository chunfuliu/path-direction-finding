<script>
    import maplibregl from "maplibre-gl";
    import PathFinder, { pathToGeoJSON } from "geojson-path-finder";
    import paths from "../data/PATH-paths.geo.json";
    import buildings from "../data/PATH-buildings.geo.json";
    import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
    import positron from "../data/positron.json";
    import { onMount } from "svelte";
    import "maplibre-gl/dist/maplibre-gl.css";

    import Link from "./Link.svelte";
    import Input from "./Input.svelte";

    let menuOpen = $state(false);
    let map;
    let startend = $state("start");
    let placeholder = $state("Find a place in PATH");
    let pathLineString;
    let start = createEmptyPoint();
    let finish = createEmptyPoint();
    let pathLocation = $state("Find a Path");
    let startValue = $state("");
    let endValue = $state("");

    function createEmptyPoint() {
        return {
            type: "Feature",
            properties: {
                Id: 0,
                BldgName: "",
                Address: null,
            },
            geometry: {
                type: "Point",
                coordinates: [], // Fresh array every time!
            },
        };
    }
    function removePathString() {
        if (pathLineString != undefined) {
            console.log(
                "removepathstring triggered, pathlinestring is not undefined",
            );
            pathLineString = undefined;
            map.removeLayer("routing-layer");
            console.log("removepathstring triggered, remove filter on start");

            map.removeSource("routing-source");

            // if we want to remove the source of routing-source, only use when we try to find new paths.
        }

        pathLineString = undefined;
    }

    function findpath(start, finish) {
        //remove source and layer if it already exists
        console.log(pathLineString);
        removePathString();

        //pathfinder function
        const pathFinder = new PathFinder(paths, {
            precision: 0.0005,
            weight: function (a, b, props) {
                // 1. Use the custom weight if it exists in your GeoJSON properties
                if (props && props.Cost) {
                    return props.Cost;
                }
            },
        });

        // 1. Calculate the raw path
        const calculatedPath = pathFinder.findPath(start, finish);

        // 2. CRITICAL SHIELD: If pathfinder returns undefined, exit gracefully instead of crashing
        if (!calculatedPath) {
            console.warn("No path could be found between these two points.");
            return;
        }

        // 3. Safe to parse now that we know it exists
        pathLineString = pathToGeoJSON(calculatedPath);

        //visualizing the information
        map.addSource("routing-source", {
            type: "geojson",
            data: pathLineString,
        });

        map.addLayer({
            id: "routing-layer",
            type: "line",
            source: "routing-source",
            paint: {
                "line-color": "#FF0000",
                "line-width": 2,
            },
        });
    }

    const menuItems = [];

    for (const feature of buildings.features) {
        let bldgname = feature.properties.BldgName.concat(
            " / (",
            feature.properties.Address.split(",")[0],
            ")",
        );
        if (feature.properties.BldgName) menuItems.push(bldgname);
        /*if (feature.properties.Address) menuItems.push(feature.properties.Address);*/
    }

    // Svelte 5 derived state: automatically updates when inputValue changes
    let filteredItemStart = $derived(
        startValue.trim() === ""
            ? []
            : menuItems.filter((item) =>
                  item.toLowerCase().includes(startValue.toLowerCase()),
              ),
    );
    let filteredItemFinish = $derived(
        endValue.trim() === ""
            ? []
            : menuItems.filter((item) =>
                  item.toLowerCase().includes(endValue.toLowerCase()),
              ),
    );

    function routePlanning(status, value, updatePoint) {
        console.log(`routePlanning triggered from ${status}`);
        map.setFilter(`paths-building-${status}`, [
            "==",
            ["get", "BldgName"],
            value.split("/")[0].trim(),
        ]);
        console.log(value.split("/")[0].trim());
        let buidingCoord = buildings.features.filter(
            (feature) =>
                feature.properties.BldgName === value.split("/")[0].trim(),
        );

        // update the coordinates of the finish point
        updatePoint.geometry.coordinates = buidingCoord[0].geometry.coordinates;
        console.log(`${status} is ${updatePoint.geometry.coordinates}`);
    }

    onMount(() => {
        map = new maplibregl.Map({
            container: "map",
            style: positron, //'https://basemaps.cartocdn.com/gl/positron-gl-style/style.json',
            center: [-79.383, 43.648], // starting position
            minZoom: 12.5,
            zoom: 13.5,
            maxZoom: 19,
            projection: "globe",
            scrollZoom: true,
            attributionControl: false,
            bearing: -15.8,
        });
        // Instantiate and add Geolocate control
        const geolocateControl = new maplibregl.GeolocateControl({
            positionOptions: {
                enableHighAccuracy: true,
            },
            trackUserLocation: true,
        });
        // Add geolocate control to the map.
        map.addControl(
            new maplibregl.GeolocateControl({
                positionOptions: {
                    enableHighAccuracy: true,
                },
                trackUserLocation: true,
            }),
        );

        map.on("load", function () {
            map.addSource("paths-source", {
                type: "geojson",
                data: paths,
            });

            map.addSource("paths-building-source", {
                type: "geojson",
                data: buildings,
            });
            /*
            map.addLayer({
                id: "paths-layer",
                type: "line",
                source: "paths-source",
                paint: {
                    "line-color": "#b6ad90",
                    "line-width": 6,
                },
            });
            */
            map.addLayer({
                id: "paths-layer",
                type: "line",
                source: "paths-source",
                paint: {
                    "line-color": [
                        "match",
                        ["get", "Level"],
                        "Level P3",
                        "#582f0e",
                        "Level P2",
                        "#7f4f24",
                        "Level P1",
                        "#936639",
                        "Level P0.5",
                        "#a68a64",
                        "Level 1",
                        "#b6ad90",
                        "Level 1.5",
                        "#c2c5aa",
                        "Level 2",
                        "#a4ac86",
                        "Level 3",
                        "#656d4a",
                        "Level 4",
                        "#414833",
                        "Level 5",
                        "#333d29",
                        "Stairs",
                        "#000000",
                        "Escalators",
                        "#4F7942",
                        "#a9d6e5",
                    ],
                    "line-width": 6,
                },
            });
            
            map.addLayer({
                id: "paths-building-layer",
                type: "circle",
                source: "paths-building-source",
                paint: {
                    "circle-radius": 5,
                    "circle-color": "#e0e1dd",
                    "circle-stroke-width": 2,
                    "circle-stroke-color": "#000000",
                },
            });

            map.addLayer({
                id: "paths-building-start",
                type: "circle",
                source: "paths-building-source",
                filter: ["==", ["get", "BldgName"], ""],
                paint: {
                    "circle-radius": 8,
                    "circle-color": "#ee9b00",
                },
            });

            map.addLayer({
                id: "paths-building-end",
                type: "circle",
                source: "paths-building-source",
                filter: ["==", ["get", "BldgName"], ""],
                paint: {
                    "circle-radius": 8,
                    "circle-color": "#AA4A44",
                },
            });
        });

        map.on("click", "paths-building-layer", function (e) {
            // 1. Get the building name from the clicked feature
            const bldgName = e.features[0].properties.BldgName;

            // 2. 💡 LOOK UP THE HIGH-PRECISION ORIGINAL COORDINATES FROM GEOMETRY IN MEMORY
            const originalBuilding = buildings.features.find(
                (f) => f.properties.BldgName === bldgName,
            );

            if (!originalBuilding) {
                console.warn("Building not found in raw data pool.");
                return;
            }
            console.log(startend);

            if (menuOpen == true) {
                startend = "end";
                map.setFilter(`paths-building-${startend}`, [
                    "==",
                    ["get", "BldgName"],
                    bldgName,
                ]);
                // 3. 💡 Use the bulletproof, unsimplified coordinates
                finish.geometry.coordinates =
                    originalBuilding.geometry.coordinates;
                findpath(start, finish);
                endValue = bldgName.concat(
                    " / (",
                    e.features[0].properties.Address.split(",")[0],
                    ")",
                );
                
                filteredItemFinish = [];
            } else if (menuOpen == false) {
                startend = "start";
                map.setFilter(`paths-building-${startend}`, [
                    "==",
                    ["get", "BldgName"],
                    bldgName,
                ]);
                removePathString();
                startValue = bldgName.concat(
                    " / (",
                    e.features[0].properties.Address.split(",")[0],
                    ")",
                );
                // 3. 💡 Use the bulletproof, unsimplified coordinates
                start.geometry.coordinates =
                    originalBuilding.geometry.coordinates;
                filteredItemStart = [];
            }
        });

        map.on("mouseenter", "paths-building-layer", () => {
            map.getCanvas().style.cursor = "pointer";
        });

        map.on("mouseleave", "paths-building-layer", () => {
            map.getCanvas().style.cursor = "";
        });
    });
</script>

<main>
    <div id="map" class="map" />
</main>

<section class="routing-panel">
    <button
        class="app-button"
        onclick={() => {
            menuOpen = !menuOpen;
            console.log(menuOpen);
            if (menuOpen == true) {
                //if in path searching mode, change search bar name to destination
                placeholder = "Start";
                pathLocation = "Find a Location"
            } else if (menuOpen == false) {
                // location finding mode
                // clear out the value in the search bar and remove the routing layers and selection on the starting building.
                removePathString();
                map.setFilter(`paths-building-end`, [
                    "==",
                    ["get", "BldgName"],
                    "",
                ]);
                placeholder = "Find a place in PATH";
                endValue = "";
                pathLocation = "Find a Path"
            }
        }}
        style="background-color: {menuOpen ? '#a9d6e5' : ''}; color: {menuOpen
            ? 'black'
            : 'black'}"
        >{pathLocation}
    </button>

    <div class="input-container">
        <Input bind:inputValue={startValue} {placeholder} />
        <!-- MENU -->
        {#if startValue.trim() !== "" && filteredItemStart.length > 0}
            <div class="dropdown-content">
                {#each filteredItemStart as item}
                    <div
                        role="button"
                        tabindex="0"
                        class="dropdown-item"
                        onclick={() => {
                            startValue = item;
                            clickedpoint = startValue;
                            filteredItemStart = [];
                            routePlanning("start", startValue, start);

                            if (finish.geometry.coordinates.length != 0) {
                                console.log("start triggers findpath");
                                findpath(start, finish);
                            }
                        }}
                        onkeydown={(e) =>
                            e.key === "Enter" && (startValue = item)}
                    >
                        <p>{item}</p>
                    </div>
                {/each}
            </div>
        {/if}
    </div>

    {#if menuOpen}
        <div class="input-container">
            <Input bind:inputValue={endValue} placeholder="Destination" />
            <!-- MENU -->
            {#if endValue.trim() !== "" && filteredItemFinish.length > 0}
                <div class="dropdown-content">
                    {#each filteredItemFinish as item}
                        <div
                            role="button"
                            tabindex="0"
                            class="dropdown-item"
                            onclick={() => {
                                endValue = item;

                                filteredItemFinish = [];

                                // if it is in seaching path mode, run the find path and route planning functions, otherwise no need to run it.
                                routePlanning("end", endValue, finish);
                                if (menuOpen) {
                                    console.log("search path mode");

                                    findpath(start, finish);
                                }
                            }}
                            onkeydown={(e) =>
                                e.key === "Enter" && (endValue = item)}
                        >
                            <p>{item}</p>
                        </div>
                    {/each}
                </div>
            {/if}
        </div>
    {/if}
</section>

<style>
    main {
        overflow-x: hidden;
    }
    .map {
        height: 100%;
        width: 100%;
        top: 0;
        left: 0%;
        position: absolute;
        overflow: hidden;
    }

    .routing-panel {
        position: absolute;
        width: auto;
        top: 2vw;
        left: 2vw;
        z-index: 1; /* Ensures it sits above the map */
        /*width: auto;  Adjust as needed */
        min-width: 250px;
        display: flex;
        flex-direction: column;
        gap: 10px; /* Space between the two search bars */
    }

    h1 {
        font-size: 18px;
        text-align: top;
    }
    .dropdown {
        position: relative;
        display: inline-block;
    }

    .dropdown-content {
        position: relative;
        background-color: #f6f6f6;
        min-width: 230px;
        max-height: 200px;
        border: 1px solid #f6f6f6;
        z-index: 1;
        font-family: Arial;
        overflow-y: auto;
    }
    .dropdown-item {
        cursor: pointer;
        min-width: 230px;
        box-sizing: border-box;
        padding-left: 15px;
        padding-right: 15px;
    }

    /* Optional: Add a hover effect so users know it's clickable */
    .dropdown-item:hover {
        background-color: #e9e9e9;
    }

    .app-button {
        font-size: 12px;
        width: auto;
        height: 28px;

        color: black;
        margin-right: 5px;
        margin-bottom: 5px;
        border-width: 0px;
        position: relative;
        font-weight: bold;
    }
</style>
