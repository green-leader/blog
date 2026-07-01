+++
date = '2026-07-01T10:59:17-06:00'
draft = false
title = 'Basic Weather'
+++
{{< output-div >}}
{{< js-inline >}}
const weatherTable = ["Sunny", "Cloudy", "Partly Cloudy", "Rainy", "Snowy", "Sleet", "Stormy", "Lightning", "Thunder", "Hail", "Windy", "Foggy", "Ice", "Tornado", "Rainbows", "Clear Sky"];

function generateWeather() {
    const weatherSelection = weatherTable[Math.floor(Math.random() * weatherTable.length)];
    return `<strong>Weather:</strong> ${weatherSelection}`;
}

document.getElementById("output").innerHTML = generateWeather();
{{< /js-inline >}}