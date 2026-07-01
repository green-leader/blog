+++
date = '2026-07-01T11:21:12-06:00'
draft = false
title = 'Mythic Fate'
+++
{{< output-div id="output" >}}
{{< output-div id="randomNumber" >}}


{{< js-inline >}}
    // Initially we just want to assume a 50/50
    // Don't really need chaos, assuming a chaos of 5

    const fateOdds = [
        ["Impossible", 1, 5, 82],
        ["No way", 3, 15, 84],
        ["Very unlikely", 5, 25, 86],
        ["Unlikely", 7, 35, 88],
        ["50/50", 10, 50, 91],
        ["Somewhat likely", 13, 65, 94],
        ["Likely", 15, 75, 96],
        ["Very likely", 16, 85, 97],
        ["Near sure thing", 18, 90, 99],
        ["A sure thing", 18, 90, 99],
        ["Has to be", 19, 95, 100]
    ];

    function generateRandomNumber() {
        const randNumber = Math.floor(Math.random() * 100);
        return `${randNumber}`;
    }

    function yesno(value, odds) {
        if (value <= odds[1]) return `Exceptional Yes`
        if (value <= odds[2]) return `Yes`;
        if (value <= odds[3]) return `No`;
        return `Exceptional No`;
    }

    function resultLoop(value, odds) {
        let resultStr = "<ul>";

        for (const element of odds) {
            let tempStr = `<li><strong>${element[0]}:</strong> ${yesno(value, element)}</li>`
            resultStr += tempStr + '\n';
        }
        resultStr += '</ul>'
        return resultStr;
    }

    const generatedNumber = generateRandomNumber();

    document.getElementById("randomNumber").innerHTML = `<strong>Rolled number:</strong> ${generatedNumber}`;
    document.getElementById("output").innerHTML = resultLoop(generatedNumber, fateOdds);
{{< /js-inline >}}