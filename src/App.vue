<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);
let dialog_closed = ref(false);
let incidents = ref([]);
let map_search_text = ref('');
let map_search_loading = ref(false);
let incidents_loading = ref(false);
let selectedMarker = null;

let filter_murder = ref(false);
let filter_robbery = ref(false);
let filter_assault = ref(false);
let filter_burglary = ref(false);
let filter_theft = ref(false);
let filter_arson = ref(false);
let filter_graffiti = ref(false);
let filter_narcotics = ref(false);

let neighborhood_1 = ref(false);
let neighborhood_2 = ref(false);
let neighborhood_3 = ref(false);
let neighborhood_4 = ref(false);
let neighborhood_5 = ref(false);
let neighborhood_6 = ref(false);
let neighborhood_7 = ref(false);
let neighborhood_8 = ref(false);
let neighborhood_9 = ref(false);
let neighborhood_10 = ref(false);
let neighborhood_11 = ref(false);
let neighborhood_12 = ref(false);
let neighborhood_13 = ref(false);
let neighborhood_14 = ref(false);
let neighborhood_15 = ref(false);
let neighborhood_16 = ref(false);
let neighborhood_17 = ref(false);



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
async function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes
    incidents_loading.value = true;
    fetch(`${crime_url.value}/incidents-expanded`)
        .then(res => res.json())
        .then(data => {
            incidents.value = data;
            console.log(incidents.value);
        })
        .catch((error) => {
            console.log('Error:', error);
        });
    incidents_loading.value = false;
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

async function deleteIncident(caseNumber) {
    if (!confirm(`Delete incident ${caseNumber}?`)) return;

    try {
        const response = await fetch(`${crime_url.value}/remove-incident`, {
            method: "DELETE",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ case_number: caseNumber })
        });

        const text = await response.text();

        if (text === "success") {
            alert("Incident deleted!");

            initializeCrimes();
        } else {
            alert("Delete failed: " + text);
        }
    } catch (err) {
        alert("Network error: " + err);
    }
}

async function selectCrime(caseNumber, block, date, time, incident) {
    // Remove old marker if it exists
    if (selectedMarker) {
        map.leaflet.removeLayer(selectedMarker);
        selectedMarker = null;
    }

    // Convert "98X UNIVERSITY AV W" → "980 UNIVERSITY AV W"
    const cleanedAddress = cleanAddress(block);

    // Geocode the address (using Nominatim)
    const geo = await fetch(`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(cleanedAddress + ", St Paul MN")}`);
    const results = await geo.json();

    if (results.length === 0) {
        alert("Could not locate address on map: " + cleanedAddress);
        return;
    }

    const { lat, lon } = results[0];

    // Custom marker icon (blue)
    const icon = L.icon({
        iconUrl: 'https://cdn-icons-png.flaticon.com/512/854/854878.png',
        iconSize: [32, 32],
        iconAnchor: [16, 32],
        popupAnchor: [0, -32]
    });

    selectedMarker = L.marker([lat, lon], { icon }).addTo(map.leaflet);

    selectedMarker.bindPopup(`
        <b>${incident}</b><br>
        Date: ${date}<br>
        Time: ${time}<br>
        Location: ${cleanedAddress}<br><br>
        <button onclick="deleteIncident('${caseNumber}')"
                style="background:#c0392b; color:white; border:none; padding:6px 12px; border-radius:5px; cursor:pointer;">
            Delete Incident
        </button>
    `).openPopup();

    map.leaflet.setView([lat, lon], 16);
}

function cleanAddress(raw) {
    let address = raw.trim();
    if (/ AND /i.test(address) || / & /i.test(address)) {
        let sep = / AND /i.test(address) ? / AND /i : / & /i;
        let [street1, street2] = address.split(sep).map(s => s.trim());
        if (!/\b(ST|AVE|AV|RD|DR|BLVD|LN|PKWY|PL|CT|WAY)\b/i.test(street2)) {
            street2 += " AVE";
        }
        address = `${street1} & ${street2}`;
    }
    let parts = address.split(" ");
    if (parts.length > 0) {
        let num = parts[0];
        if (/^\d+X$/.test(num)) {
            num = num.replace("X", "0");
        } else if (/^\d+X+$/.test(num)) {
            num = num.replace(/X+/g, match => "0".repeat(match.length));
        }
        parts[0] = num;
    }
    const dirMap = { N:"NORTH", S:"SOUTH", E:"EAST", W:"WEST" };
    parts = parts.map(tok => dirMap[tok.toUpperCase()] || tok);
    parts = parts.map(tok => {
        if (/^(ST|MT|FT)[A-Z]{2,}/.test(tok)) {
            return tok.slice(0, tok.match(/^(ST|MT|FT)/)[0].length) + " " + tok.slice(tok.match(/^(ST|MT|FT)/)[0].length);
        }
        return tok;
    });

    return parts.join(" ");
}

async function submitForm (e) {
    // event automatically prevented by @submit.prevent

    const msg = document.getElementById("form-message");
    msg.textContent = "";

    // Collect form fields
    const data = {
        case_number: document.getElementById("f-case").value.trim(),
        date: document.getElementById("f-date").value,
        time: document.getElementById("f-time").value,
        code: document.getElementById("f-code").value,
        incident: document.getElementById("f-incident").value.trim(),
        police_grid: document.getElementById("f-grid").value,
        neighborhood_number: document.getElementById("f-hood").value,
        block: document.getElementById("f-block").value.trim()
    };

    // Validate (simple)
    for (let key in data) {
        if (!data[key]) {
            msg.style.color = "red";
            msg.textContent = `Error: Missing required field "${key}"`;
            return;
        }
    }

    // PUT request
    try {
        const res = await fetch(`${crime_url.value}/new-incident`, {
            method: "PUT",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(data)
        });

        if (!res.ok) {
            const err = await res.json();
            msg.style.color = "red";
            msg.textContent = "Upload failed: " + err.error;
            return;
        }

        msg.style.color = "green";
        msg.textContent = "Incident successfully added!";

        // Refresh data on map + table
        const crimeRes = await fetch('/incidents-expanded');
        incidents = await crimeRes.json();
        initializeCrimes();

        e.target.reset();

    } catch (err) {
        msg.style.color = "red";
        msg.textContent = "Network error: " + err;
    }
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

function applyFilters() {
    let codes = [];
    if (filter_murder.value) {
        for (let i = 100; i < 200; i++)
            codes.push(i);
    }
    if (filter_robbery.value) {
        for (let i = 300; i < 400; i++)
            codes.push(i);
    }
    if (filter_assault.value) {
        for (let i = 400; i < 500; i++)
            codes.push(i);
    }
    if (filter_burglary.value) {
        for (let i = 500; i < 600; i++)
            codes.push(i);
    }
    if (filter_theft.value) {
        for (let i = 600; i < 800; i++)
            codes.push(i);
    }
    if (filter_arson.value) {
        for (let i = 900; i < 1000; i++)
            codes.push(i);
    }
    if (filter_graffiti.value) {
        for (let i = 1400; i < 1500; i++)
            codes.push(i);
    }
    if (filter_narcotics.value) {
        for (let i = 1800; i < 1900; i++)
            codes.push(i);
    }
    if (codes.length !== 0) {
        codes = 'code='+codes.toString();
    } else {
        codes = '';
    }

    let neighborhoods = [];
    if (neighborhood_1.value) neighborhoods.push(1);
    if (neighborhood_2.value) neighborhoods.push(2);
    if (neighborhood_3.value) neighborhoods.push(3);
    if (neighborhood_4.value) neighborhoods.push(4);
    if (neighborhood_5.value) neighborhoods.push(5);
    if (neighborhood_6.value) neighborhoods.push(6);
    if (neighborhood_7.value) neighborhoods.push(7);
    if (neighborhood_8.value) neighborhoods.push(8);
    if (neighborhood_9.value) neighborhoods.push(9);
    if (neighborhood_10.value) neighborhoods.push(10);
    if (neighborhood_11.value) neighborhoods.push(11);
    if (neighborhood_12.value) neighborhoods.push(12);
    if (neighborhood_13.value) neighborhoods.push(13);
    if (neighborhood_14.value) neighborhoods.push(14);
    if (neighborhood_15.value) neighborhoods.push(15);
    if (neighborhood_16.value) neighborhoods.push(16);
    if (neighborhood_17.value) neighborhoods.push(17);
    if (neighborhoods.length !== 0) {
        neighborhoods = 'neighborhood_number='+neighborhoods.toString();
    } else {
        neighborhoods = '';
    }

    

    console.log(codes);
    
    incidents_loading.value = true;
    fetch(`${crime_url.value}/incidents-expanded?${codes}&${neighborhoods}`)
        .then(res => res.json())
        .then(data => {
            incidents.value = data;
            console.log(incidents.value);
        })
        .catch((error) => {
            console.log('Error:', error);
        });
    incidents_loading.value = false;
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
            <div class="cell auto small-12">
                <a :href="crime_url+'/map.html'" target="_self"><button>Map V2 (neighborhood markers and filtering)</button></a> 
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
                    <form id="incident-form" @submit.prevent="submitForm">
                        
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
                        <p>Incident Type</p>
                        <input type="checkbox" v-model="filter_murder"/>
                            <label for="checkbox">Murder</label><br>
                        <input type="checkbox" v-model="filter_robbery"/>
                            <label for="checkbox">Robbery</label><br>
                        <input type="checkbox" v-model="filter_assault"/>
                            <label for="checkbox">Assault</label><br>
                        <input type="checkbox" v-model="filter_burglary"/>
                            <label for="checkbox">Burglary</label><br>
                        <input type="checkbox" v-model="filter_theft"/>
                            <label for="checkbox">Theft</label><br>
                        <input type="checkbox" v-model="filter_arson"/>
                            <label for="checkbox">Arson</label><br>
                        <input type="checkbox" v-model="filter_graffiti"/>
                            <label for="checkbox">Graffiti</label><br>
                        <input type="checkbox" v-model="filter_narcotics"/>
                            <label for="checkbox">Narcotics</label><br><br>


                        <p>Neighborhood Name</p>
                        <input type="checkbox" v-model="neighborhood_1"/>
                            <label for="checkbox">Conway/Battlecreek/Highwood</label><br>
                        <input type="checkbox" v-model="neighborhood_2"/>
                            <label for="checkbox">Greater East Side</label><br>
                        <input type="checkbox" v-model="neighborhood_3"/>
                            <label for="checkbox">West Side</label><br>
                        <input type="checkbox" v-model="neighborhood_4"/>
                            <label for="checkbox">Dayton's Bluff</label><br>
                        <input type="checkbox" v-model="neighborhood_5"/>
                            <label for="checkbox">Payne/Phalen</label><br>
                        <input type="checkbox" v-model="neighborhood_6"/>
                            <label for="checkbox">North End</label><br>
                        <input type="checkbox" v-model="neighborhood_7"/>
                            <label for="checkbox">Thomas/Dale (Frogtown)</label><br>
                        <input type="checkbox" v-model="neighborhood_8"/>
                            <label for="checkbox">Summit/University</label><br>
                        <input type="checkbox" v-model="neighborhood_9"/>
                            <label for="checkbox">West Seventh</label><br>
                        <input type="checkbox" v-model="neighborhood_10"/>
                            <label for="checkbox">Como</label><br>
                        <input type="checkbox" v-model="neighborhood_11"/>
                            <label for="checkbox">Hamline/Midway</label><br>
                        <input type="checkbox" v-model="neighborhood_12"/>
                            <label for="checkbox">St. Anthony</label><br>
                        <input type="checkbox" v-model="neighborhood_13"/>
                            <label for="checkbox">Union Park</label><br>
                        <input type="checkbox" v-model="neighborhood_14"/>
                            <label for="checkbox">Macalester-Groveland</label><br>
                        <input type="checkbox" v-model="neighborhood_15"/>
                            <label for="checkbox">Highland</label><br>
                        <input type="checkbox" v-model="neighborhood_16"/>
                            <label for="checkbox">Summit Hill</label><br>
                        <input type="checkbox" v-model="neighborhood_17"/>
                            <label for="checkbox">Capitol River</label><br><br>
                        
                        <label for="date">Start Date<br>
                            <input type="date" v-model="filter_start"/>
                        </label>
                        <label for="date">End Date<br>
                            <input type="date" v-model="filter_end"/>
                        </label>
                        <label for="number">Max Incidents<br>
                            <input type="number" v-model="filter_limit"/>
                        </label>

                        <button type="button" @click="applyFilters">Apply Filters</button>
                        
                    </div>
                </div>
            </div>


        <!-- Incident Table -->
        <div class="custom-card grid-x grid-padding-x grid-padding-y">
            <div class="cell auto small-12">
                <h2>Crimes in Visible Neighborhoods</h2>
                <h2 v-if="incidents_loading">loading...</h2>
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
                    <tr v-for="inc in incidents" :style="categorizeCrime(inc.incident_type)" @click="selectCrime(inc.case_number, inc.block, inc.date, inc.time, inc.incident_type)">
                        <td>{{ inc.case_number }}</td>
                        <td>{{ inc.date }}</td>
                        <td>{{ inc.time }}</td>
                        <td>{{ inc.code }}</td>
                        <td>{{ inc.incident_type }}</td>
                        <td>{{ inc.neighborhood_name }}</td>
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
