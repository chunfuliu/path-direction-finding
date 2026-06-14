<script>
    import maplibregl from "maplibre-gl";
    import PathFinder, { pathToGeoJSON } from "geojson-path-finder";
    import paths from "../data/PATH-paths.geo.json";
    import buildings from "../data/PATH-buildings.geo.json";
    import * as turf from "@turf/turf"; // this is for fitting the map boundary to GTA municipalities
    import { difference } from "@turf/difference";
    import { polygon } from "@turf/helpers";
    import positron from "../data/positron.json";
    import { onMount } from "svelte";
    import "maplibre-gl/dist/maplibre-gl.css";


    import Input from "./Input.svelte";

    let menuOpen = $state(false);
    let map;
    let startend = $state("start");
    let placeholder = $state("Type here or click on map to find a place in PATH");
    let pathLineString;
    let start = createEmptyPoint();
    let finish = createEmptyPoint();
    let pathLocation = $state("Click to Find a Path");
    let startValue = $state("");
    let endValue = $state("");
    let colouring =  
        [
            "match",
            ["get", "Level"],
            "Level P3",                         "#9b2226",
            "Level P2",                         "#2596be",
            "Level P1",                         "#72C3B8",
            "Level P0.5",                       "#6C7678",
            "Level 1",                          "#00B6E6",
            "Level 1.5",                        "#0a9396",
            "Level 2",                          "#005f73",
            "Level 3",                          "#e09f3e",
            "Level 4",                          "#b08968",
            "Level 5",                          "#333d29",
            "Stairs",                           "#588157",
            "Escalators",                       "#F49200",
            "Slope",                            "#C3B1E1",
            "Elevator",                         "#B4C424",
            "Sky Bridge",                       "#7393B3",
            "",                                 "red",
            "Level 1 (Outside)",                "#6699CD",
            "#a9d6e5",
        ]

    function createEmptyPoint() {
        return {
            type: "Feature",
            properties: {
                Id: 0,
                BldgName: "",
                Address: "",
            },
            geometry: {
                type: "Point",
                coordinates: [], // Fresh array every time!
            },
        };
    }
    function removePathString() {
        if (pathLineString != undefined) {

            pathLineString = undefined;
            map.removeLayer("routing-layer");

            map.removeSource("routing-source");

            map.removeLayer("routing-mask-layer");

            map.removeSource("routing-mask-source");

            map.removeLayer("routing-line-layer");

            map.removeSource("routing-line-source");

            // if we want to remove the source of routing-source, only use when we try to find new paths.
        }
        pathLineString = undefined;
    }

    function findpath(start, finish) {
        //remove source and layer if it already exists

        removePathString();
        //pathfinder function
        const pathFinder = new PathFinder(paths, {
            precision: 0.0000005,
            weight: function (a, b, props) {
                // Use the custom weight if it exists in your GeoJSON properties
                if (props && props.Cost) {
                    if (props.Level == "Elevator" || props.Level == "Escalator"){
                        return props.Cost
                    }
                    else {
                        const dx = a[0] - b[0]
                        const dy = a[1] - b[1]
                        return Math.sqrt(dx * dx + dy * dy)*props.Cost;
                    }
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

        let coordinates = pathLineString.geometry.coordinates
        // Pass the first coordinates in the LineString to `lngLatBounds` &
        // wrap each coordinate pair in `extend` to include them in the bounds
        // result. A variation of this technique could be applied to zooming
        // to the bounds of multiple Points or Polygon geometries - it just
        // requires wrapping all the coordinates with the extend method.
        const bounds = coordinates.reduce((bounds, coord) => {
            return bounds.extend(coord);
        }, new maplibregl.LngLatBounds(coordinates[0], coordinates[0]));

        map.fitBounds(
            bounds,
            { padding: {
                top: 200,
                left: 50,
                right: 50,
                bottom: 50,
            },
            bearing: map.getBearing()
        },
        );
        var bufferedmask = turf.buffer(pathLineString, 1000, { units: "meters" });
        var buffered = turf.buffer(pathLineString, 2, { units: "meters" });
        
        //cutout the mask
        const erasedResult = difference({
            type: 'FeatureCollection',
            features: [bufferedmask, buffered]
            });


        //visualizing the information
        map.addSource("routing-mask-source", {
            type: "geojson",
            data: erasedResult,
        });

        map.addLayer({
            id: "routing-mask-layer",
            type: "fill",
            source: "routing-mask-source",
            paint: {
                "fill-opacity": 0.2,
                "fill-color": "black"
            },
        });

        //visualizing the information
        map.addSource("routing-source", {
            type: "geojson",
            data: buffered,
        });

        map.addLayer({
            id: "routing-layer",
            type: "line",
            source: "routing-source",
            paint: {
                "line-color": "yellow",
                "line-width": {
                        "stops": [
                        [
                            15,
                            1
                        ],
                        [
                            18,
                            5
                        ]
                    
                    ]
                },
            },
        });

        //visualizing the information
        map.addSource("routing-line-source", {
            type: "geojson",
            data: pathLineString,
        });

        map.addLayer({
            id: "routing-line-layer",
            type: "line",
            source: "routing-line-source",
            paint: {
                "line-color": "yellow",
                "line-width": {
                        "stops": [
                        [
                            15,
                            1
                        ],
                        [
                            18,
                            1.5
                        ]
                    
                    ]
                }
            },
        });
        map.moveLayer('poi-paths-building-labels')
        map.moveLayer('paths-building-layer')
        map.moveLayer('paths-building-end')
        map.moveLayer('paths-building-start')
        
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
            zoom: 14.5,
            maxZoom: 22,
            projection: "globe",
            scrollZoom: true,
            attributionControl: false,
            bearing: -15.8,
        });

        const geolocateControl = new maplibregl.GeolocateControl({
            positionOptions: {
                enableHighAccuracy: true,
            },
            trackUserLocation: true,
        });
        map.addControl(geolocateControl);

        const userLocationMarkerEl = document.createElement("div");
        userLocationMarkerEl.className = "user-arrow";
        userLocationMarkerEl.innerHTML = `
            <svg viewBox="0 0 24 24" width="28" height="28" xmlns="http://www.w3.org/2000/svg">
                <path d="M12 2 L19 21 L12 17 L5 21 Z" fill="#FF5722" stroke="#3E2723" stroke-width="1" />
            </svg>
        `;
        userLocationMarkerEl.style.transformOrigin = "center center";

        const userLocationMarker = new maplibregl.Marker({ element: userLocationMarkerEl });

        geolocateControl.on("geolocate", (event) => {
            const { longitude, latitude, heading } = event.coords || event;
            userLocationMarker.setLngLat([longitude, latitude]).addTo(map);

            const angle = heading !== null && heading !== undefined ? heading : map.getBearing();
            if (angle !== undefined && angle !== null) {
                userLocationMarkerEl.style.transform = `rotate(${angle}deg)`;
            }
        });

        map.on("load", function () {
            map.addSource("paths-source", {
                type: "geojson",
                data: paths,
            });

            map.addSource("paths-building-source", {
                type: "geojson",
                data: buildings,
            });
            
            map.addLayer({
                id: "paths-layer",
                type: "line",
                source: "paths-source",
                filter: ["match", ["get", "Level"], 
                        ["Level P3",        "Level P2",             
                        "Level P1",         "Level P0.5",        
                        "Level 1",          "Level 1.5",          
                        "Level 2",          "Level 3",         
                        "Level 4",          "Level 5", 
                        "Sky Bridge",       "Elevator", "" ],
                        true,  // Return true if matched (keep the feature)
                        false
                    ],
                paint: {
                    "line-color": colouring,
                    "line-width": {
                        "stops": [
                        [
                            16,
                            3
                        ],
                        [
                            18,
                            10
                        ]
                    
                    ]
                }
                },
            });
            
            map.addLayer({
                id: "paths-dashed-layer",
                type: "line",
                source: "paths-source",
                filter: ["match", ["get", "Level"], 
                        ["Stairs",      "Escalators",   
                        "Level 1 (Outside)",          
                         "Slope"      ],
                        true,  // Return true if matched (keep the feature)
                        false
                    ],
                paint: {
                    "line-color": colouring,
                    "line-width": {
                        "stops": [
                        [
                            15,
                            3
                        ],
                        [
                            18,
                            6
                        ]
                    
                    ]
                },
                    'line-dasharray': [1, 0.75],
                },
            });

            map.addLayer({
                id: 'poi-labels',
                type: 'symbol',
                source: 'paths-source',
                
                layout: {
                    'text-field': ['get', 'Level'],
                    'text-font': ['Helvetica'],
                    "text-offset": [0, 1],
                    "text-rotation-alignment": "map",
                    
                    'text-justify': 'auto',
                    "symbol-placement": "line",
                    "symbol-spacing": 200,
                    "text-size": 12
                }, 
                minzoom: 15,
                paint: {
                    "text-halo-color": "white",
                    "text-halo-width": 2,
                    "text-color": "#484848",
                    
                }
            });
            map.addLayer({
                id: "paths-building-layer",
                type: "circle",
                source: "paths-building-source",
                paint: {
                    "circle-radius": 5,
                    "circle-color": "#e0e1dd",
                    "circle-stroke-width": 2,
                    "circle-stroke-color": "#585858",
                },
            });

            map.addLayer({
                id: "paths-building-start",
                type: "circle",
                source: "paths-building-source",
                filter: ["==", ["get", "BldgName"], ""],
                paint: {
                    "circle-radius": 5,
                    "circle-color": "#ff9f1c",
                    "circle-stroke-width": 2,
                    "circle-stroke-color": "#000000"
                },
            });

            map.addLayer({
                id: "paths-building-end",
                type: "circle",
                source: "paths-building-source",
                filter: ["==", ["get", "BldgName"], ""],
                paint: {
                    "circle-radius": 5,
                    "circle-color": "#AA4A44",
                    "circle-stroke-width": 2,
                    "circle-stroke-color": "#000000"
                },
            });

            map.addLayer({
                id: 'poi-paths-building-labels',
                type: 'symbol',
                source: 'paths-building-source',
                minzoom: 16,
                layout: {
                    'text-field': ['get', 'BldgName'],
                    'text-font': ['Helvetica Bold'],
                    "text-anchor": "top",
                    "text-rotation-alignment": "viewport",
                    /*'text-radial-offset': 0.5,*/
                    "text-offset": [0, 0.5],
                    'text-justify': 'center',
                    "text-max-width": 8,
                    "text-size": 12
                }, 
                minZoom: 20,
                paint: {
                    "text-halo-color": "white",
                    "text-halo-width": 2,
                    "text-color": "#484848",
                    
                }
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
                pathLocation = "Click to Find a Location"
            } else if (menuOpen == false) {
                // location finding mode
                // clear out the value in the search bar and remove the routing layers and selection on the starting building.
                removePathString();
                map.setFilter(`paths-building-end`, [
                    "==",
                    ["get", "BldgName"],
                    "",
                ]);
                placeholder = "Click to Find a place in PATH";
                endValue = "";
                pathLocation = "Click to Find a Path"
            }
        }}
        style="background-color: {menuOpen ? '#283618' : '#F49200'}; color: {menuOpen
            ? 'white'
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
                        
                        class="dropdown-item"
                        onclick={() => {
                            startValue = item;
                            filteredItemStart = [];
                            console.log("clicked on the start box")
                            routePlanning("start", startValue, start);

                            if (finish.geometry.coordinates.length != 0) {
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
            <Input bind:inputValue={endValue} placeholder="Type or Click on map to find a destination" />
            <!-- MENU -->
            {#if endValue.trim() !== "" && filteredItemFinish.length > 0}
                <div class="dropdown-content">
                    {#each filteredItemFinish as item}
                        <div
                            role="button"
                            
                            class="dropdown-item"
                            onclick={() => {
                                endValue = item;

                                filteredItemFinish = [];

                                // if it is in seaching path mode, run the find path and route planning functions, otherwise no need to run it.
                                routePlanning("end", endValue, finish);
                                if (menuOpen) {
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
        width: 40vw;
        top: 2vh;
        left: 2vw;
        background-color: #588157;
        padding: 10px;
        z-index: 1; /* Ensures it sits above the map */
        display: flex;
        flex-direction: column;
        gap: 10px; /* Space between the two search bars */
        border-radius: 10px;
        max-width: calc(100vw - 4vw);
        box-sizing: border-box;

    }



    .dropdown-content {
        position: relative;
        background-color: #f6f6f6;
        min-width: 230px;
        max-height: 200px;
        width: auto;
        border: 1px solid #f6f6f6;
        z-index: 1;
        font-family: Verdana;
        overflow-y: auto;
        border-radius: 5px;
        outline: 3px solid #f6bd60;
        box-sizing: border-box;
        max-width: 100%;
    }
    .dropdown-item {
        cursor: pointer;
        min-width: 230px;
        box-sizing: border-box;
        padding-left: 15px;
        padding-right: 15px;
        border-radius: 20px;
        
    }

    /* Optional: Add a hover effect so users know it's clickable */
    .dropdown-item:hover {
        background-color: #e9e9e9;
    }

    .app-button {
        font-size: 12px;
        width: auto;
        height: 28px;
        border-radius: 20px;
        color: black;
        border-width: 0px;
        position: relative;
        font-weight: bold;
    }

    .user-arrow {
        width: 28px;
        height: 28px;
        display: flex;
        align-items: center;
        justify-content: center;
        transform-origin: center center;
        transition: transform 150ms linear;
        pointer-events: none;
    }

    .user-arrow svg path {
        stroke: #00000022;
    }

    .maplibregl-user-location-dot,
    .maplibregl-user-location-accuracy-circle {
        display: none !important;
    }

    /* --- Tablets (600px and up) --- */
@media screen and (max-width: 900px) {
  .routing-panel {
    width: calc(90vw - 4vw);
    left: 2vw;
    right: 2vw;
    min-width: unset;
  }
}

</style>
