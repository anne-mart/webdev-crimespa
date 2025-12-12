<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);
let dialog_closed = ref(false);
let incidents = ref([]);
let map_search_text = ref('');
let map_search_loading = ref(false);

let selected_marker = ref(null);

let allCrimes = [];
let something = null;

let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null},
            {location: [44.977413, -93.025156], marker: null},
            {location: [44.931244, -93.079578], marker: null},
            {location: [44.956192, -93.060189], marker: null},
            {location: [44.978883, -93.068163], marker: null},
            {location: [44.975766, -93.113887], marker: null},
            {location: [44.959639, -93.121271], marker: null},
            {location: [44.947700, -93.128505], marker: null},
            {location: [44.930276, -93.119911], marker: null},
            {location: [44.982752, -93.147910], marker: null},
            {location: [44.963631, -93.167548], marker: null},
            {location: [44.973971, -93.197965], marker: null},
            {location: [44.949043, -93.178261], marker: null},
            {location: [44.934848, -93.176736], marker: null},
            {location: [44.913106, -93.170779], marker: null},
            {location: [44.937705, -93.136997], marker: null},
            {location: [44.949203, -93.093739], marker: null}
        ]
    }
);

//let markers = L.layerGroup().addTo(map);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 17
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
    });
});


// FUNCTIONS
// Function called once user has entered REST API URL
function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
    fetch(`${crime_url.value}/incidents-expanded`)
        .then(res => res.json())
        .then(data => {
            incidents.value = data;
            console.log(incidents.value);
        })
        .catch((error) => {
            console.log('Error:', error);
        });
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        dialog_closed.value = true;
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

function categorizeCrime(type) {
    const t = type.toUpperCase();

    // Violent crimes ... light red
    if (t.includes("HOMICIDE") ||
        t.includes("MURDER") ||
        t.includes("ROBBERY") ||
        t.includes("ASSAULT") ||
        t.includes("SEX") ||
        t.includes("RAPE") ||
        t.includes("KIDNAP")) {
        return { backgroundColor: '#ff000022' };
    }
    // Property crimes... light blue
    if (t.includes("BURGLARY") ||
        t.includes("THEFT") ||
        t.includes("LARCENY") ||
        t.includes("VANDALISM") ||
        t.includes("DAMAGE") ||
        t.includes("ARSON") ||
        t.includes("FRAUD") ||
        t.includes("FORGERY")) {
        return { backgroundColor: '#0000ff22' };
    }

    // Other... light green
    return { backgroundColor: '#00aa0022' };
}

function deleteIncident(case_number) {
    console.log("Deleting incident: " + case_number);
    if (!confirm(`Delete incident ${case_number}?`)) return;

    fetch(`${crime_url.value}/remove-incident`, {
        method: "DELETE",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ case_number: case_number })
    })
        .then(res => {
            if (res.status === 200) {
                alert("Incident deleted!");

                initializeCrimes();
            } else {
                alert("Delete failed: " + res.statusText);
            }
        })
        .catch((error) => {
            console.log('Error:', error);
        });
}


function searchMap() {
    map_search_loading.value = true;
    let url = `https://nominatim.openstreetmap.org/search?q=${map_search_text.value}&format=jsonv2&limit=1`;
    fetch(url)
        .then((response) => {
            return response.json();
        })
        .then((data) => {
            // zooms the map to the bounding area of the location entered, based on the Nominatim response
            map.leaflet.fitBounds([[data[0].boundingbox[0], data[0].boundingbox[2]], [data[0].boundingbox[1], data[0].boundingbox[3]]]);
            map_search_loading.value = false;
        })
}

</script>

<template>

    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>

    

    <div class="grid-container">
        <div class="grid-x grid-padding-x grid-padding-y">  
            <div class="cell auto small-12">
                <h1>St. Paul Crime Incidents</h1>
            </div>
            <div class="cell auto small-12">
                <a :href="crime_url+'/about.html'" target="_self"><button>About</button></a> 
            </div>
        </div>

        <!-- Incident Map-->
        <div class="custom-card grid-x grid-padding-x grid-padding-y">
            <div class="cell auto small-12">
                <h2>St. Paul Crime Map</h2>
            </div>
            <div id="leafletmap" class="cell auto small-12 large-6"></div>
            <div id="searchmap" class="cell auto small-12 large-6">
                <h2>Search Map</h2>
                <input type="text" placeholder="enter latitude & longitude coordinates, address, etc." v-model="map_search_text"></input>
                <button type="button" @click="searchMap">Go</button>
                <h1 v-if="map_search_loading">loading...</h1>
            </div>
        </div>


        
            <div class="grid-x grid-padding-x grid-padding-y align-justify two-col-row">
                <!-- Incident Form -->
                <div class="custom-card cell auto small-12 large-6">
                    <h2>Submit New Crime Incident</h2>
                    <form id="incident-form">
                        
                        <label>Case Number (required)<br>
                            <input id="f-case" required>
                        </label><br>

                        <label>Date (YYYY-MM-DD) (required)<br>
                            <input id="f-date" type="date" required>
                        </label><br>

                        <label>Time (HH:MM) (required)<br>
                            <input id="f-time" type="time" required>
                        </label><br>

                        <label>Crime Code (required)<br>
                            <input id="f-code" type="number" required>
                        </label><br>

                        <label>Incident Type (required)<br>
                            <input id="f-incident" required>
                        </label><br>

                        <label>Police Grid (required)<br>
                            <input id="f-grid" type="number" required>
                        </label><br>

                        <label>Neighborhood # (required)<br>
                            <input id="f-hood" type="number" required>
                        </label><br>

                        <label>Block (required)<br>
                            <input id="f-block" required>
                        </label><br>

                        <button type="submit" style="padding:8px 18px;">Submit</button>
                    </form>

                    <div id="form-message" style="font-weight:bold; margin-bottom:20px;"></div>

                </div>

                <!-- Incident Filters -->
                <div class="custom-card cell auto small-12 large-6" style="margin-left:auto;">
                    <h2>Filters</h2>
                    <div>
                        <ul>
                            <li>TEST</li>
                            <li>TEST</li>
                            <li>TEST</li>
                        </ul>
                    </div>
                </div>
            </div>


        <!-- Incident Table -->
        <div class="custom-card grid-x grid-padding-x grid-padding-y">
            <div class="cell auto small-12">
                <h2>Crimes in Visible Neighborhoods <span id="count">(loading...)</span></h2>
            </div>

            <div id="legend">
                <div class="legend-item"><div class="legend-color" style="background:#ff000022;"></div>Violent Crimes</div>
                <div class="legend-item"><div class="legend-color" style="background:#0000ff22;"></div>Property Crimes</div>
                <div class="legend-item"><div class="legend-color" style="background:#00aa0022;"></div>Other Crimes</div>
            </div>

            <table>
                <thead>
                    <tr>
                        <th>Case #</th>
                        <th>Date</th>
                        <th>Time</th>
                        <th>Code</th>
                        <th>Type</th>
                        <th>Neighborhood</th>
                        <th>Block</th>
                        <th>Delete</th>
                    </tr>
                </thead>
                <tbody id="crime-tbody">
                    <tr v-for="inc in incidents" :style="categorizeCrime(inc.incident_type)">
                        <td>{{ inc.case_number }}</td>
                        <td>{{ inc.date }}</td>
                        <td>{{ inc.time }}</td>
                        <td>{{ inc.code }}</td>
                        <td>{{ inc.type }}</td>
                        <td>{{ inc.neighborhood }}</td>
                        <td>{{ inc.block }}</td>
                        <td><button class="delete" @click.stop="deleteIncident(inc.case_number)">Delete</button></td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
    
    margin: 0;
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: #c2cdd9;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}

h2 {
    font-size: 2rem;
    color:#34495e;
}


        body { margin:0; padding:20px; font-family:Arial,sans-serif; background:#f8f9fa; }
        #leafletmap {  border:3px solid #2c3e50; border-radius:10px; }
        h1 { text-align:center; color:#2c3e50; margin-bottom:10px; }
        table { width:100%; border-collapse:collapse; background:white; box-shadow:0 3px 10px rgba(0,0,0,0.1); }
        th, td { padding:10px; border:1px solid #ddd; text-align:left; }
        th { background:#2c3e50; color:white; }
        #count { font-weight:bold; color:#d31a3f; }

        #legend {
            margin: 15px 0;
            padding: 10px 15px;
            background: white;
            border: 1px solid #ccc;
            border-radius: 8px;
            width: fit-content;
            box-shadow: 0 2px 6px rgba(0,0,0,0.15);
        }
        .legend-item {
            display:flex;
            align-items:center;
            margin-bottom:6px;
            font-size:14px;
        }
        .legend-color {
            width:18px;
            height:18px;
            border-radius:4px;
            margin-right:8px;
            border:1px solid #888;
        }
        
button {
    border: solid 1px #000000;
    background: white;
    width: 100%;
    border-radius:10px;
    height: 39px;
    color: black;
}
button:hover {
    background: #34495e;
    color: white;
}

.custom-card {
    background-color: #c2cdd9;
    border-radius: 20px;
    padding: 20px;
    margin-bottom: 30px;
}

@media screen and (min-width: 64em) {
    .two-col-row {
        gap: 20px;
    }
    .two-col-row > .cell.large-6 {
        flex: 0 0 calc(50% - 10px);
        max-width: calc(50% - 10px);
    }
}

.delete {
    background:#c0392b; 
    color:white; 
    border:none; 
    padding:6px 12px; 
    border-radius:5px; 
    cursor:pointer;
}

</style>
