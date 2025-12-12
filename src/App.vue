<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);
let dialog_closed = ref(false);
let show_filters = ref(false);
let incidents = ref([]);

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

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
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
    fetch(`${crime_url.value}/incidents`)
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

function showHideFilters() {
    show_filters.value = !show_filters.value;
    console.log(show_filters.value);
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

    <div  class="grid-container">
        <div class="grid-x grid-padding-x grid-padding-y">  
            <h1>St. Paul Crime Incidents</h1>
        </div>

        <!-- Incident Filters -->
        <div class="grid-x grid-padding-x grid-padding-y">
            <div id="filters" class="cell auto small-12">
                <h2>Filters</h2>
                <button type="button" @click="showHideFilters" v-if="show_filters">Hide Filters ^</button>
                <button type="button" @click="showHideFilters" v-if="!show_filters">Show Filters v</button>
                <div v-if="show_filters">
                    <ul>
                        <li>TEST</li>
                        <li>TEST</li>
                        <li>TEST</li>
                    </ul>
                </div>
            </div>
        </div>

        <!-- Map & Table -->
        <div class="grid-x grid-padding-x grid-padding-y">
            <div class="cell auto small-12">
                <h2>Map</h2>
            </div>
            <div id="leafletmap" class="cell auto small-12 large-6"></div>
            <div id="" class="cell auto small-12 large-6">
                <h2>TEST</h2>
            </div>
        </div>


        <div class="grid-x grid-padding-x grid-padding-y">
            <div class="cell auto small-12">
                <h2>Incidents Table</h2>
            </div>

            <!-- Remove Incident Section -->
            <div class="cell auto small-12">
                <label for="case_input">Case Number to Remove:</label>
                <input id="case_input" type="text" placeholder="Enter case number">
                <button id="remove-btn" type="button">Remove Incident</button>
                <p id="remove-result"></p>
            </div>

            <!-- Incident Table -->
            <div class="cell auto small-12">
                <table>
                    <thead>
                        <tr>
                            <th>Case #</th>
                            <th>Date</th>
                            <th>Time</th>
                            <th>Code</th>
                            <th>Incident</th>
                            <th>Grid</th>
                            <th>Neighborhood</th>
                            <th>Block</th>
                        </tr>
                    </thead>
                    <tbody id="incident-tbody">
                        <tr v-for="inc in incidents">
                            <td>{{inc.case_number}}</td>
                            <td>{{inc.date}}</td>
                            <td>{{inc.time}}</td>
                            <td>{{inc.code}}</td>
                            <td>{{inc.incident}}</td>
                            <td>{{inc.police_grid}}</td>
                            <td>{{inc.neighborhood_number}}</td>
                            <td>{{inc.block}}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
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
    color: #8923bf;
}
</style>
