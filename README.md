<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Interactive Route Booking</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .header {
            width: 100%;
            background-color: #333;
            color: white;
            padding: 15px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .header button {
            background: none;
            border: none;
            color: white;
            font-size: 1em;
            cursor: pointer;
            padding: 10px;
        }

        .container {
            width: 90%;
            max-width: 600px;
            padding: 20px;
            background: white;
            box-shadow: 0px 0px 15px rgba(0, 0, 0, 0.1);
            border-radius: 8px;
            margin-top: 20px;
            text-align: center;
        }

        .input-group {
            position: relative;
            margin-bottom: 20px;
        }

        .input-group input {
            width: 100%;
            padding: 10px;
            font-size: 1em;
            border: 1px solid #ccc;
            border-radius: 5px;
        }

        .suggestions {
            position: absolute;
            width: 100%;
            background: white;
            border: 1px solid #ccc;
            border-top: none;
            z-index: 100;
            max-height: 150px;
            overflow-y: auto;
        }

        .suggestion-item {
            padding: 10px;
            cursor: pointer;
        }

        .suggestion-item:hover {
            background-color: #f0f0f0;
        }

        #map {
            width: 100%;
            height: 400px;
            margin-top: 20px;
            border-radius: 8px;
        }

        #busOptions {
            margin-top: 20px;
            text-align: left;
        }

        .bus-option {
            padding: 10px;
            background: #e9ecef;
            margin-bottom: 10px;
            border-radius: 5px;
        }
    </style>
</head>
<body>

    <!-- Header Section -->
    <div class="header">
        <button>Home</button>
        <button>Profile</button>
        <button>Ride History</button>
    </div>

    <div class="container">
        <h2>Book Your Route</h2>

        <div class="input-group">
            <input type="text" id="departure" placeholder="Enter departure location" autocomplete="off">
            <div id="departureSuggestions" class="suggestions"></div>
        </div>

        <div class="input-group">
            <input type="text" id="destination" placeholder="Enter destination" autocomplete="off">
            <div id="destinationSuggestions" class="suggestions"></div>
        </div>

        <div id="map"></div>

        <div id="busOptions" style="display: none;">
            <h3>Available Buses:</h3>
        </div>
    </div>

    <script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
    <script src="https://unpkg.com/leaflet-routing-machine/dist/leaflet-routing-machine.min.js"></script>
    <script>
        const departureInput = document.getElementById("departure");
        const destinationInput = document.getElementById("destination");
        const departureSuggestions = document.getElementById("departureSuggestions");
        const destinationSuggestions = document.getElementById("destinationSuggestions");
        const busOptions = document.getElementById("busOptions");

        let departureCoords, destinationCoords, map, routeControl;

        // Initialize the map
        map = L.map('map').setView([20, 0], 2);
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            maxZoom: 18,
        }).addTo(map);

        // Fetch suggestions for locations
        async function fetchSuggestions(query, callback) {
            if (query.length < 1) return;
            const response = await fetch(`https://api.opencagedata.com/geocode/v1/json?q=${encodeURIComponent(query)}&key=bf01552c80824841bb8f1773332f6a4a`);
            const data = await response.json();
            callback(data.results);
        }

        // Show location suggestions
        function showSuggestions(suggestions, targetElement) {
            targetElement.innerHTML = '';
            suggestions.forEach(result => {
                const item = document.createElement('div');
                item.classList.add('suggestion-item');
                item.textContent = result.formatted;
                item.addEventListener('click', () => selectLocation(result, targetElement));
                targetElement.appendChild(item);
            });
        }

        // Select location from suggestions
        function selectLocation(result, targetElement) {
            if (targetElement === departureSuggestions) {
                departureInput.value = result.formatted;
                departureCoords = result.geometry;
            } else if (targetElement === destinationSuggestions) {
                destinationInput.value = result.formatted;
                destinationCoords = result.geometry;
            }
            targetElement.innerHTML = '';
            drawRoute();
        }

        // Draw the route on the map
        function drawRoute() {
            if (departureCoords && destinationCoords) {
                if (routeControl) map.removeControl(routeControl);
                routeControl = L.Routing.control({
                    waypoints: [
                        L.latLng(departureCoords.lat, departureCoords.lng),
                        L.latLng(destinationCoords.lat, destinationCoords.lng)
                    ],
                    createMarker: function(i, waypoint, n) {
                        return L.marker(waypoint.latLng);
                    },
                    routeWhileDragging: false,
                    show: false
                }).addTo(map);

                routeControl.on('routesfound', async function(e) {
                    displayBusOptions(e.routes[0].coordinates);
                });
            }
        }

        // Fetch meaningful location name and filter out unnamed roads
        async function fetchLocationName(coord) {
            const response = await fetch(`https://api.opencagedata.com/geocode/v1/json?q=${coord.lat},${coord.lng}&key=4d681936f8484d12ae976f3d4a2336f9`);
            const data = await response.json();
            if (data.results && data.results[0]) {
                // Filter out locations with "Unnamed Road"
                const formatted = data.results[0].formatted;
                if (formatted && !formatted.toLowerCase().includes("unnamed road")) {
                    return formatted;
                }
            }
            return null; // If it's an unnamed road or no valid location, return null
        }

        async function displayBusOptions(routeCoordinates) {
            busOptions.innerHTML = '<h3>Available Buses:</h3>';
            busOptions.style.display = 'block';

            const busRoutes = ['Bus 101', 'Bus 202', 'Bus 303', 'Bus 404', 'Bus 505'];
            const intermediateStops = {};

            for (const bus of busRoutes) {
                intermediateStops[bus] = [];
                for (let i = 1; i < routeCoordinates.length - 1; i += Math.floor(routeCoordinates.length / 6)) {
                    const coord = routeCoordinates[i];
                    const locationName = await fetchLocationName(coord);
                    if (locationName && !intermediateStops[bus].includes(locationName)) {
                        intermediateStops[bus].push(locationName);
                    }
                    if (intermediateStops[bus].length >= 6) break; // Limit to 6 stops per bus
                }
            }

            // Display bus options with named stops
            for (const bus in intermediateStops) {
                const busOption = document.createElement('div');
                busOption.classList.add('bus-option');
                busOption.innerHTML = `
                    <strong>${bus}</strong> - via: ${intermediateStops[bus].join(', ')}
                    <button onclick="bookBus('${bus}')">Book Now</button>
                `;
                busOptions.appendChild(busOption);
            }
        }

        function bookBus(bus) {
            alert(`You have selected ${bus}. Your booking is being processed.`);
        }

        // Attach event listeners for input fields
        departureInput.addEventListener('input', () => fetchSuggestions(departureInput.value, suggestions => showSuggestions(suggestions, departureSuggestions)));
        destinationInput.addEventListener('input', () => fetchSuggestions(destinationInput.value, suggestions => showSuggestions(suggestions, destinationSuggestions)));
    </script>
</body>
</html>
