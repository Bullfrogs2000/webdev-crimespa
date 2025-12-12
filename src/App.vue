
<script setup>
import { reactive, ref, onMounted } from 'vue'

/**
 * Reactive map state and constants
 */
const crime_url = ref('');
const baseApiUrl = ref('');
const dialog_err = ref(false);

const map = reactive({
    leaflet: null,
    center: {
        lat: 44.955139,
        lng: -93.102222,
        address: ''
    },
    zoom: 12,
    bounds: {
        // Bounding box for St. Paul
        nw: { lat: 45.008206, lng: -93.217977 },
        se: { lat: 44.883658, lng: -92.993787 }
    }
});

// Approximate neighborhood marker locations (1..17)
const neighborhoodMarkerCoords = [
    [44.942068, -93.020521],
    [44.977413, -93.025156],
    [44.931244, -93.079578],
    [44.956192, -93.060189],
    [44.978883, -93.068163],
    [44.975766, -93.113887],
    [44.959639, -93.121271],
    [44.947700, -93.128505],
    [44.930276, -93.119911],
    [44.982752, -93.147910],
    [44.963631, -93.167548],
    [44.973971, -93.197965],
    [44.949043, -93.178261],
    [44.934848, -93.176736],
    [44.913106, -93.170779],
    [44.937705, -93.136997],
    [44.949203, -93.093739]
];

/**
 * Crime data and filters
 */
const codes = ref([]); // raw codes from API
const incidentTypeGroups = ref([]); // [{ type, codes:[], selected }]
const neighborhoods = ref([]); // [{ id, name, lat, lng, marker, crimes, selected }]
const incidents = ref([]); // processed incidents for table

const visibleNeighborhoodIds = ref([]); // neighborhood ids currently within map bounds

const isLoadingCrimes = ref(false);
const loadError = ref('');

const filters = reactive({
    dateStart: '',
    dateEnd: '',
    maxIncidents: 1000
});

const locationQuery = ref(''); // address / lat,lng input

// new incident form
const newIncident = reactive({
    case_number: '',
    date: '',
    time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: ''
});
const newIncidentError = ref('');
const newIncidentSuccess = ref('');

// extra UI state
const selectedCaseNumber = ref(null);
const selectedIncidentMarker = ref(null);

// cache for geocoding crime blocks
const geocodeCache = reactive({});

const NOMINATIM_BASE = 'https://nominatim.openstreetmap.org';

/**
 * Vue lifecycle
 */
onMounted(() => {
    // Create Leaflet map (set bounds and valid zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);

    map.leaflet.setMaxBounds([
        [map.bounds.se.lat, map.bounds.nw.lng],
        [map.bounds.nw.lat, map.bounds.se.lng]
    ]);

    // Draw neighborhood boundaries from GeoJSON
    const district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
        .then((response) => response.json())
        .then((result) => {
            result.features.forEach((value) => {
                district_boundary.addData(value);
            });
        })
        .catch((error) => {
            console.log('Error loading GeoJSON:', error);
        });

    // Update when user pans/zooms
    map.leaflet.on('moveend', handleMapMoveEnd);
});

/**
 * Helper: clamp lat/lng to St. Paul bounding box
 */
function clampLatLng(lat, lng) {
    const minLat = map.bounds.se.lat;
    const maxLat = map.bounds.nw.lat;
    const minLng = map.bounds.nw.lng;
    const maxLng = map.bounds.se.lng;

    const clampedLat = Math.min(Math.max(lat, minLat), maxLat);
    const clampedLng = Math.min(Math.max(lng, minLng), maxLng);
    return { lat: clampedLat, lng: clampedLng };
}

/**
 * Initialize codes, neighborhoods, and default crimes once REST URL is provided
 */
async function initializeCrimes() {
    loadError.value = '';
    isLoadingCrimes.value = false;

    const trimmed = crime_url.value.trim();
    if (!trimmed) {
        loadError.value = 'You must enter a valid REST API URL.';
        return;
    }
    baseApiUrl.value = trimmed.replace(/\/+$/, '');

    try {
        await loadCodes();
        await loadNeighborhoods();
        updateVisibleNeighborhoods();
        await fetchIncidents();
    } catch (err) {
        console.error(err);
        loadError.value = 'Failed to load codes/neighborhoods/incidents. Check REST API URL and server.';
    }
}

/**
 * Load /codes and build type groups
 */
async function loadCodes() {
    const resp = await fetch(`${baseApiUrl.value}/codes`);
    if (!resp.ok) {
        throw new Error('Failed to load crime codes');
    }
    const data = await resp.json();
    codes.value = data;

    const groupsMap = new Map();
    for (const c of data) {
        if (!groupsMap.has(c.type)) {
            groupsMap.set(c.type, { type: c.type, codes: [], selected: true });
        }
        groupsMap.get(c.type).codes.push(c.code);
    }
    incidentTypeGroups.value = Array.from(groupsMap.values()).sort((a, b) =>
        a.type.localeCompare(b.type)
    );
}

/**
 * Load /neighborhoods and create markers
 */
async function loadNeighborhoods() {
    const resp = await fetch(`${baseApiUrl.value}/neighborhoods`);
    if (!resp.ok) {
        throw new Error('Failed to load neighborhoods');
    }
    const data = await resp.json();

    neighborhoods.value = data.map((n, idx) => {
        const coords = neighborhoodMarkerCoords[idx] || [map.center.lat, map.center.lng];
        let marker = null;
        if (map.leaflet) {
            marker = L.circleMarker(coords, {
                radius: 7,
                weight: 1,
                fillOpacity: 0.7
            }).addTo(map.leaflet);
            marker.bindPopup(`${n.name}: 0 crimes`);
        }
        return {
            id: n.id,
            name: n.name,
            lat: coords[0],
            lng: coords[1],
            marker,
            crimes: 0,
            selected: true
        };
    });
}

/**
 * Determine which neighborhoods are visible in map viewport
 */
function updateVisibleNeighborhoods() {
    if (!map.leaflet || neighborhoods.value.length === 0) return;

    const bounds = map.leaflet.getBounds();
    const north = bounds.getNorth();
    const south = bounds.getSouth();
    const west = bounds.getWest();
    const east = bounds.getEast();

    const visible = neighborhoods.value
        .filter((n) => n.lat <= north && n.lat >= south && n.lng >= west && n.lng <= east)
        .map((n) => n.id);

    visibleNeighborhoodIds.value = visible;
}

/**
 * Fetch incidents with current filters and map view
 */
async function fetchIncidents() {
    if (!baseApiUrl.value) return;
    isLoadingCrimes.value = true;
    loadError.value = '';

    try {
        // Build code filter from selected incident types
        const selectedTypes = incidentTypeGroups.value
            .filter((g) => g.selected)
            .map((g) => g.type);

        let selectedCodes = [];
        if (selectedTypes.length > 0) {
            const codeSet = new Set();
            for (const g of incidentTypeGroups.value) {
                if (selectedTypes.includes(g.type)) {
                    g.codes.forEach((c) => codeSet.add(c));
                }
            }
            selectedCodes = Array.from(codeSet.values()).sort((a, b) => a - b);
        }

        // Neighborhood filter: combine selected checkboxes + visible neighborhoods
        let selectedNeighborhoodIds = neighborhoods.value
            .filter((n) => n.selected)
            .map((n) => n.id);

        if (selectedNeighborhoodIds.length === 0) {
            // if user unchecked everything, treat as "none"
            selectedNeighborhoodIds = [];
        }

        // if user has some selected, base set is those; otherwise all neighborhoods
        const baseNeighborhoodIds =
            selectedNeighborhoodIds.length > 0
                ? selectedNeighborhoodIds
                : neighborhoods.value.map((n) => n.id);

        const inViewIds =
            visibleNeighborhoodIds.value.length > 0
                ? baseNeighborhoodIds.filter((id) => visibleNeighborhoodIds.value.includes(id))
                : baseNeighborhoodIds.slice();

        if (inViewIds.length === 0) {
            incidents.value = [];
            updateNeighborhoodCountsFromIncidents();
            isLoadingCrimes.value = false;
            return;
        }

        const params = new URLSearchParams();
        params.set('limit', String(filters.maxIncidents > 0 ? filters.maxIncidents : 1000));

        if (filters.dateStart) params.set('start_date', filters.dateStart);
        if (filters.dateEnd) params.set('end_date', filters.dateEnd);

        if (selectedCodes.length > 0) {
            params.set('code', selectedCodes.join(','));
        }

        if (inViewIds.length > 0) {
            params.set('neighborhood', inViewIds.join(','));
        }

        const url = `${baseApiUrl.value}/incidents?${params.toString()}`;
        const resp = await fetch(url);
        if (!resp.ok) {
            const text = await resp.text();
            throw new Error(text || 'Failed to fetch incidents');
        }
        const data = await resp.json();

        // Build helper maps
        const codeToType = new Map();
        for (const c of codes.value) {
            codeToType.set(c.code, c.type);
        }
        const neighborhoodMap = new Map();
        for (const n of neighborhoods.value) {
            neighborhoodMap.set(n.id, n.name);
        }

        // Process incidents
        incidents.value = data.map((row) => {
            const incident_type = codeToType.get(row.code) || 'Unknown';
            const neighborhood_name = neighborhoodMap.get(row.neighborhood_number) || 'Unknown';
            const category = categorizeCrime(incident_type, row.incident);
            return {
                ...row,
                incident_type,
                neighborhood_name,
                category
            };
        });

        updateNeighborhoodCountsFromIncidents();
    } catch (err) {
        console.error(err);
        loadError.value = err.message || 'Failed to fetch incidents';
    } finally {
        isLoadingCrimes.value = false;
    }
}

/**
 * Update per-neighborhood crime counts and popups
 */
function updateNeighborhoodCountsFromIncidents() {
    const counts = new Map();
    for (const n of neighborhoods.value) {
        counts.set(n.id, 0);
    }
    for (const crime of incidents.value) {
        if (counts.has(crime.neighborhood_number)) {
            counts.set(crime.neighborhood_number, counts.get(crime.neighborhood_number) + 1);
        }
    }
    neighborhoods.value.forEach((n) => {
        n.crimes = counts.get(n.id) || 0;
        if (n.marker) {
            n.marker.bindPopup(
                `${n.name}: ${n.crimes} crime${n.crimes === 1 ? '' : 's'}`
            );
        }
    });
}

/**
 * Categorize crime into violent / property / other
 */
function categorizeCrime(incidentType, incidentText) {
    const text = `${incidentType} ${incidentText || ''}`.toLowerCase();

    const violentKeywords = [
        'murder',
        'manslaughter',
        'rape',
        'robbery',
        'assault',
        'kidnap',
        'homicide',
        'weapon',
        'threat',
        'domestic'
    ];

    const propertyKeywords = [
        'burglary',
        'theft',
        'robbery',
        'arson',
        'vandalism',
        'damage to property',
        'property',
        'motor vehicle theft',
        'shoplifting',
        'stolen',
        'fraud',
        'forgery'
    ];

    if (violentKeywords.some((k) => text.includes(k))) {
        return 'violent';
    }
    if (propertyKeywords.some((k) => text.includes(k))) {
        return 'property';
    }
    return 'other';
}

/**
 * Dialog handlers
 */
function closeDialog() {
    const dialog = document.getElementById('rest-dialog');
    const url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    } else {
        dialog_err.value = true;
    }
}

/**
 * Map move handler: update center, address, visible neighborhoods, and crimes
 */
async function handleMapMoveEnd() {
    if (!map.leaflet) return;
    const center = map.leaflet.getCenter();
    map.center.lat = center.lat;
    map.center.lng = center.lng;
    await updateLocationFromLatLng(center.lat, center.lng);
    updateVisibleNeighborhoods();
    if (baseApiUrl.value) {
        await fetchIncidents();
    }
}

/**
 * Location search / Go button
 */
async function goToLocation() {
    if (!map.leaflet) return;
    const query = locationQuery.value.trim();
    if (!query) return;

    // Try lat,lng first
    const latLngMatch = query.match(
        /^(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)$/
    );
    try {
        if (latLngMatch) {
            let lat = parseFloat(latLngMatch[1]);
            let lng = parseFloat(latLngMatch[2]);
            const clamped = clampLatLng(lat, lng);
            lat = clamped.lat;
            lng = clamped.lng;
            map.leaflet.setView([lat, lng], map.leaflet.getZoom());
            await updateLocationFromLatLng(lat, lng);
        } else {
            // Treat as address
            const addrQuery = `${query}, Saint Paul, MN`;
            const url = `${NOMINATIM_BASE}/search?format=json&q=${encodeURIComponent(
                addrQuery
            )}&limit=1&addressdetails=1`;
            const resp = await fetch(url, {
                headers: { Accept: 'application/json' }
            });
            if (!resp.ok) throw new Error('Address lookup failed');
            const data = await resp.json();
            if (Array.isArray(data) && data.length > 0) {
                let lat = parseFloat(data[0].lat);
                let lng = parseFloat(data[0].lon);
                const clamped = clampLatLng(lat, lng);
                lat = clamped.lat;
                lng = clamped.lng;
                map.leaflet.setView([lat, lng], map.leaflet.getZoom());
                await updateLocationFromLatLng(lat, lng);
            } else {
                alert('Location not found within St. Paul.');
            }
        }
    } catch (err) {
        console.error(err);
        alert('Error looking up location.');
    }
}

/**
 * Reverse geocode map center and update input
 */
async function updateLocationFromLatLng(lat, lng) {
    map.center.lat = lat;
    map.center.lng = lng;
    try {
        const url = `${NOMINATIM_BASE}/reverse?format=json&lat=${lat}&lon=${lng}&zoom=17&addressdetails=1`;
        const resp = await fetch(url, {
            headers: { Accept: 'application/json' }
        });
        if (resp.ok) {
            const data = await resp.json();
            if (data && data.display_name) {
                map.center.address = data.display_name;
                locationQuery.value = data.display_name;
                return;
            }
        }
    } catch (err) {
        console.error('Reverse geocode error', err);
    }
    // fallback to coordinates
    locationQuery.value = `${lat.toFixed(5)}, ${lng.toFixed(5)}`;
}

/**
 * New incident form submission
 */
async function submitNewIncident() {
    newIncidentError.value = '';
    newIncidentSuccess.value = '';

    // ensure baseApiUrl & codes loaded
    if (!baseApiUrl.value) {
        newIncidentError.value = 'Enter REST API URL first.';
        return;
    }

    const requiredFields = [
        'case_number',
        'date',
        'time',
        'code',
        'incident',
        'police_grid',
        'neighborhood_number',
        'block'
    ];
    for (const field of requiredFields) {
        if (!String(newIncident[field]).trim()) {
            newIncidentError.value = 'All fields are required.';
            return;
        }
    }

    const payload = {
        case_number: String(newIncident.case_number).trim(),
        date: newIncident.date,
        time: newIncident.time,
        code: Number(newIncident.code),
        incident: String(newIncident.incident).trim(),
        police_grid: Number(newIncident.police_grid),
        neighborhood_number: Number(newIncident.neighborhood_number),
        block: String(newIncident.block).trim()
    };

    try {
        const resp = await fetch(`${baseApiUrl.value}/new-incident`, {
            method: 'PUT',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });
        if (!resp.ok) {
            const text = await resp.text();
            throw new Error(text || 'Failed to add incident');
        }
        newIncidentSuccess.value = 'Incident successfully added.';
        await fetchIncidents();
    } catch (err) {
        console.error(err);
        newIncidentError.value = err.message || 'Failed to add incident.';
    }
}

/**
 * Delete incident (from table or popup)
 */
async function deleteIncident(incident) {
    if (!baseApiUrl.value) return;
    const confirmation = confirm(
        `Delete incident ${incident.case_number}? This cannot be undone.`
    );
    if (!confirmation) return;

    try {
        const resp = await fetch(`${baseApiUrl.value}/remove-incident`, {
            method: 'DELETE',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ case_number: incident.case_number })
        });
        if (!resp.ok) {
            const text = await resp.text();
            throw new Error(text || 'Failed to delete incident');
        }
        if (selectedCaseNumber.value === incident.case_number) {
            selectedCaseNumber.value = null;
            if (selectedIncidentMarker.value) {
                map.leaflet.removeLayer(selectedIncidentMarker.value);
                selectedIncidentMarker.value = null;
            }
        }
        await fetchIncidents();
    } catch (err) {
        console.error(err);
        alert(err.message || 'Failed to delete incident.');
    }
}

/**
 * Normalize block address for geocoding (handle 9X formats)
 */
function normalizeBlockAddress(block) {
    let addr = String(block || '').trim();

    // Replace a single leading house-number X with 0 (e.g. 98X -> 980)
    const m = addr.match(/^(\d+X)\s+(.*)$/i);
    if (m) {
        const number = m[1].replace('X', '0');
        addr = `${number} ${m[2]}`;
    }
    return `${addr}, Saint Paul, MN`;
}

/**
 * Geocode incident block, cache results, and return lat/lng
 */
async function geocodeBlock(block) {
    if (!block) return null;
    if (geocodeCache[block]) return geocodeCache[block];

    const query = normalizeBlockAddress(block);
    const url = `${NOMINATIM_BASE}/search?format=json&q=${encodeURIComponent(
        query
    )}&limit=1`;

    try {
        const resp = await fetch(url, { headers: { Accept: 'application/json' } });
        if (!resp.ok) throw new Error('Geocoding failed');
        const data = await resp.json();
        if (Array.isArray(data) && data.length > 0) {
            let lat = parseFloat(data[0].lat);
            let lng = parseFloat(data[0].lon);
            const clamped = clampLatLng(lat, lng);
            geocodeCache[block] = clamped;
            return clamped;
        }
    } catch (err) {
        console.error('Geocode error for block', block, err);
    }
    return null;
}

/**
 * Select a crime from the table and show marker
 */
async function selectIncident(incident) {
    selectedCaseNumber.value = incident.case_number;
    if (!map.leaflet) return;

    const coords = await geocodeBlock(incident.block);
    if (!coords) {
        alert('Could not determine exact location for this incident.');
        return;
    }

    if (!selectedIncidentMarker.value) {
        selectedIncidentMarker.value = L.marker([coords.lat, coords.lng]).addTo(
            map.leaflet
        );
    } else {
        selectedIncidentMarker.value.setLatLng([coords.lat, coords.lng]);
    }

    map.leaflet.setView([coords.lat, coords.lng], Math.max(map.leaflet.getZoom(), 14));

    const popupId = `popup-delete-${incident.case_number}`;
    const popupHtml = `
        <div>
            <strong>${incident.incident}</strong><br/>
            ${incident.date} ${incident.time}<br/>
            ${incident.block}<br/>
            <button id="${popupId}" type="button">Delete</button>
        </div>
    `;
    selectedIncidentMarker.value
        .bindPopup(popupHtml)
        .openPopup();

    // Attach click handler for delete button inside popup
    setTimeout(() => {
        const btn = document.getElementById(popupId);
        if (btn) {
            btn.onclick = (e) => {
                e.preventDefault();
                deleteIncident(incident);
            };
        }
    }, 0);
}

/**
 * Apply filter changes via "Update" button
 */
async function applyFilters() {
    await fetchIncidents();
}

/**
 * Helper for row CSS class
 */
function crimeRowClass(crime) {
    const base =
        crime.category === 'violent'
            ? 'crime-row-violent'
            : crime.category === 'property'
            ? 'crime-row-property'
            : 'crime-row-other';
    return [
        base,
        selectedCaseNumber.value === crime.case_number ? 'selected-row' : ''
    ].join(' ');
}
</script>

<template>
    <!-- REST URL dialog -->
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input
            id="dialog-url"
            class="dialog-input"
            type="url"
            v-model="crime_url"
            placeholder="http://localhost:8000"
        />
        <p class="dialog-error" v-if="dialog_err || loadError">
            Error: {{ loadError || 'must enter valid URL' }}
        </p>
        <br />
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>

    <div class="grid-container">
        <div class="grid-x grid-padding-x top-section">
            <div class="cell medium-8 small-12">
                <div id="leafletmap"></div>

                <!-- location controls -->
                <div class="location-controls">
                    <label class="dialog-label" for="location-input">
                        Map Location (address or lat,lng):
                    </label>
                    <div class="location-input-row">
                        <input
                            id="location-input"
                            class="dialog-input"
                            v-model="locationQuery"
                            type="text"
                            placeholder="e.g. 44.95, -93.10 or 15 W Kellogg Blvd"
                        />
                        <button class="button tiny" type="button" @click="goToLocation">
                            Go
                        </button>
                    </div>
                    <p class="location-note">
                        Panning or zooming the map will update this field when the move ends.
                    </p>
                </div>
            </div>

            <!-- filters + new incident form -->
            <div class="cell medium-4 small-12 side-panel">
                <!-- Filters -->
                <fieldset class="filters">
                    <legend>Crime Filters</legend>

                    <div class="filter-group">
                        <label>Date range:</label>
                        <div class="date-range">
                            <input type="date" v-model="filters.dateStart" />
                            <span>to</span>
                            <input type="date" v-model="filters.dateEnd" />
                        </div>
                    </div>

                    <div class="filter-group">
                        <label for="max-incidents">Max incidents:</label>
                        <input
                            id="max-incidents"
                            type="number"
                            min="1"
                            v-model.number="filters.maxIncidents"
                        />
                    </div>

                    <div class="filter-group">
                        <label>Incident types:</label>
                        <div class="checkbox-list">
                            <label
                                v-for="group in incidentTypeGroups"
                                :key="group.type"
                                class="checkbox-item"
                            >
                                <input type="checkbox" v-model="group.selected" />
                                {{ group.type }}
                            </label>
                        </div>
                    </div>

                    <div class="filter-group">
                        <label>Neighborhoods:</label>
                        <div class="checkbox-list">
                            <label
                                v-for="n in neighborhoods"
                                :key="n.id"
                                class="checkbox-item"
                            >
                                <input type="checkbox" v-model="n.selected" />
                                {{ n.name }}
                            </label>
                        </div>
                        <p class="filter-note">
                            Only incidents in neighborhoods currently visible on the map will
                            be shown.
                        </p>
                    </div>

                    <button class="button small" type="button" @click="applyFilters">
                        Update Crimes
                    </button>

                    <p v-if="isLoadingCrimes" class="loading-text">Loading crimes...</p>
                </fieldset>

                <!-- Legend -->
                <div class="legend">
                    <strong>Legend:</strong>
                    <div class="legend-row">
                        <span class="legend-item">
                            <span class="legend-swatch violent"></span> Violent crimes
                        </span>
                        <span class="legend-item">
                            <span class="legend-swatch property"></span> Property crimes
                        </span>
                        <span class="legend-item">
                            <span class="legend-swatch other"></span> Other crimes
                        </span>
                    </div>
                </div>

                <!-- New incident form -->
                <fieldset class="new-incident-form">
                    <legend>New Incident</legend>

                    <label>Case number:</label>
                    <input v-model="newIncident.case_number" type="text" />

                    <label>Date:</label>
                    <input v-model="newIncident.date" type="date" />

                    <label>Time:</label>
                    <input v-model="newIncident.time" type="time" />

                    <label>Incident (description):</label>
                    <input v-model="newIncident.incident" type="text" />

                    <label>Code:</label>
                    <select v-model="newIncident.code">
                        <option disabled value="">Select a code</option>
                        <option
                            v-for="c in codes"
                            :key="c.code"
                            :value="c.code"
                        >
                            {{ c.code }} - {{ c.type }}
                        </option>
                    </select>

                    <label>Police grid:</label>
                    <input v-model="newIncident.police_grid" type="number" />

                    <label>Neighborhood:</label>
                    <select v-model="newIncident.neighborhood_number">
                        <option disabled value="">Select a neighborhood</option>
                        <option
                            v-for="n in neighborhoods"
                            :key="n.id"
                            :value="n.id"
                        >
                            {{ n.id }} - {{ n.name }}
                        </option>
                    </select>

                    <label>Block (address):</label>
                    <input
                        v-model="newIncident.block"
                        type="text"
                        placeholder="e.g. 98X UNIVERSITY AV W"
                    />

                    <p v-if="newIncidentError" class="dialog-error">
                        {{ newIncidentError }}
                    </p>
                    <p v-if="newIncidentSuccess" class="dialog-success">
                        {{ newIncidentSuccess }}
                    </p>

                    <button class="button small" type="button" @click="submitNewIncident">
                        Submit Incident
                    </button>
                </fieldset>
            </div>
        </div>

        <!-- Crimes table -->
        <div class="grid-x grid-padding-x">
            <div class="cell small-12">
                <h2>Crime Incidents (most recent first)</h2>
                <table class="crimes-table">
                    <thead>
                        <tr>
                            <th>Delete</th>
                            <th>Case #</th>
                            <th>Date</th>
                            <th>Time</th>
                            <th>Incident</th>
                            <th>Incident Type</th>
                            <th>Neighborhood</th>
                            <th>Police Grid</th>
                            <th>Block</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr
                            v-for="crime in incidents"
                            :key="crime.case_number"
                            :class="crimeRowClass(crime)"
                            @click="selectIncident(crime)"
                        >
                            <td>
                                <button
                                    class="button tiny alert"
                                    type="button"
                                    @click.stop="deleteIncident(crime)"
                                >
                                    Delete
                                </button>
                            </td>
                            <td>{{ crime.case_number }}</td>
                            <td>{{ crime.date }}</td>
                            <td>{{ crime.time }}</td>
                            <td>{{ crime.incident }}</td>
                            <td>{{ crime.incident_type }}</td>
                            <td>{{ crime.neighborhood_name }}</td>
                            <td>{{ crime.police_grid }}</td>
                            <td>{{ crime.block }}</td>
                        </tr>
                        <tr v-if="!isLoadingCrimes && incidents.length === 0">
                            <td colspan="9" style="text-align: center;">
                                No incidents match the current filters or map view.
                            </td>
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
    font-size: 0.9rem;
    color: #d32323;
}

.dialog-success {
    font-size: 0.9rem;
    color: #1f7a1f;
}

.top-section {
    margin-top: 1rem;
}

.location-controls {
    margin-top: 0.75rem;
    padding: 0.5rem;
    background-color: #f8f8f8;
    border-radius: 4px;
}

.location-input-row {
    display: flex;
    gap: 0.5rem;
    margin-top: 0.25rem;
}

.location-note {
    font-size: 0.8rem;
    color: #555;
    margin-top: 0.25rem;
}

.side-panel {
    margin-top: 1rem;
}

.filters,
.new-incident-form {
    background-color: #f9f9f9;
    padding: 0.5rem 0.75rem;
    margin-bottom: 0.75rem;
    border-radius: 4px;
    border: 1px solid #ddd;
}

.filter-group {
    margin-bottom: 0.5rem;
}

.date-range {
    display: flex;
    align-items: center;
    gap: 0.25rem;
}

.checkbox-list {
    max-height: 8rem;
    overflow-y: auto;
    border: 1px solid #ddd;
    padding: 0.25rem;
    background-color: #fff;
}

.checkbox-item {
    display: block;
    font-size: 0.85rem;
}

.filter-note {
    font-size: 0.75rem;
    color: #555;
    margin-top: 0.25rem;
}

.loading-text {
    font-size: 0.85rem;
    margin-top: 0.25rem;
}

.legend {
    margin-bottom: 0.75rem;
}

.legend-row {
    display: flex;
    flex-wrap: wrap;
    gap: 0.75rem;
    margin-top: 0.25rem;
}

.legend-item {
    display: inline-flex;
    align-items: center;
    font-size: 0.85rem;
}

.legend-swatch {
    width: 1rem;
    height: 1rem;
    border: 1px solid #555;
    margin-right: 0.25rem;
}

.legend-swatch.violent {
    background-color: #ffe5e5;
}

.legend-swatch.property {
    background-color: #e5f1ff;
}

.legend-swatch.other {
    background-color: #e9ffe5;
}

.crimes-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 0.5rem;
    font-size: 0.85rem;
}

.crimes-table th,
.crimes-table td {
    border: 1px solid #ccc;
    padding: 0.25rem 0.4rem;
}

.crimes-table thead {
    background-color: #f0f0f0;
}

.crimes-table tbody tr:hover {
    filter: brightness(0.97);
    cursor: pointer;
}

.crime-row-violent {
    background-color: #ffe5e5;
}

.crime-row-property {
    background-color: #e5f1ff;
}

.crime-row-other {
    background-color: #e9ffe5;
}

.selected-row {
    outline: 2px solid #333;
}
</style>
