+++
date = '2026-07-01T13:06:00-06:00'
draft = false
title = 'Random IP'
+++
Generate a random IP address with a subnet mask from 0 to 33.
After doing the subnet math, click the link to check your work.

{{< output-div >}}
{{< js-inline >}}
    function generateRandomIP() {
        const octets = Array.from({ length: 4 }, () => Math.floor(Math.random() * 256));
        const ip = octets.join(".");
        const cidr = Math.floor(Math.random() * 33);
        // https://www.calculator.net/ip-subnet-calculator.html?cclass=any&csubnet=${cidr}&cip=${ip}&ctype=ipv4&printit=0&x=94&y=15
        return `<a href="https://www.calculator.net/ip-subnet-calculator.html?cclass=any&csubnet=${cidr}&cip=${ip}&ctype=ipv4&printit=0&x=94&y=15">${ip}/${cidr}</a>`;
    }

    document.getElementById("output").innerHTML = generateRandomIP();
{{< /js-inline >}}