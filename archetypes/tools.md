+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
+++
{{< output-div >}}
{{< js-inline >}}
    function greet() {
        return `Hello World!`;
    }
    document.getElementById("output").innerHTML = greet();
{{< /js-inline >}}