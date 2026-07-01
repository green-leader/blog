+++
date = '2026-07-01T10:48:25-06:00'
draft = false
title = 'Advanced Weather'
+++
{{< output-div >}}


Random Starting weather, then we advance to weather that's more likely to happen next via steps.

{{< js-inline >}}
const weatherTable = ["Clear", "Partly Cloudy", "Mostly Cloudy", "Cloudy", "Precipitation", ];

function setCookie(cname, cvalue, exdays) {
    const d = new Date();
    d.setTime(d.getTime() + (exdays * 24 * 60 * 60 * 1000));
    let expires = "expires=" + d.toUTCString();
    if (typeof exdays !== "undefined") {
        document.cookie = cname + "=" + cvalue + ";" + expires + ";path=/" + ";SameSite=Strict";
    } else {
        document.cookie = cname + "=" + cvalue + ";" + ";path=/" + ";SameSite=Strict";
    }
}

function getCookie(cname) {
    let name = cname + "=";
    let decodedCookie = decodeURIComponent(document.cookie);
    let ca = decodedCookie.split(';');
    for (let i = 0; i < ca.length; i++) {
        let c = ca[i];
        while (c.charAt(0) == ' ') {
            c = c.substring(1);
        }
        if (c.indexOf(name) == 0) {
            return c.substring(name.length, c.length);
        }
    }
    return "";
}

function advanceWeather(weatherVar) {
    if (weatherVar >= weatherTable.length - 1) {
        weatherVar = -1;
    }
    return ++weatherVar;
}

function generateCookieWeather() {
    let weather = Math.floor(Math.random() * weatherTable.length);
    if (getCookie("weather") != "") {
        weather = getCookie("weather");
    }
    const weatherSelection = weather;
    setCookie("weather", advanceWeather(weather));
    const severity = Math.floor(Math.random() * 8) + 1;
    return `<strong>Weather:</strong> ${severity === 8 ? "Severe" : "" } ${weatherTable[weatherSelection]}`;
}

document.getElementById("output").innerHTML = generateCookieWeather();
{{< /js-inline >}}