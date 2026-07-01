+++
date = '2026-07-01T13:09:50-06:00'
draft = false
title = 'URL Adjust'
+++

Sometimes adjusting URLs sucks.

Rather opinionated as well.

{{< form-field copy=true >}}

{{< radio-button name=markdown checked=true >}}
{{< radio-button name=html >}}
{{< radio-button name=raw >}}

{{< checkbox name="strip params" id="strip-params" checked=true >}}

{{< output-div id=output code=true >}}

{{< js-inline >}}
    function copyFunction() {
        var copyText = document.getElementById("output");
        navigator.clipboard.writeText(copyText.innerText);
    }

    function stripParams(url) {
    const result = url.split('?')[0];
    return result;
    }

    function formatMarkdown(url) {
    let linkText = url.replace(/\:\/\/www./i, "\:\/\/")
    .replace(/https\:\/\/|http\:\/\//i, "")
    .replace(/reddit.com|old.reddit.com/i, "");
    linkText = stripParams(linkText);
    // [example.com](https://example.com)
    let returnValue = "[" + (linkText) + "](" + (url) + ")";
    document.getElementById("output").innerHTML = returnValue;
    }

    function formatHTML(url) {
    let linkText = url.replace(/\:\/\/www./i, "\:\/\/")
    .replace(/https\:\/\/|http\:\/\//i, "")
    .replace(/reddit.com|old.reddit.com/i, "");
    linkText = stripParams(linkText);
    // <a href="https://www.example.com/">example.com</a> 
    let returnValue = "&lt;a href=\"" + url + "\"&gt;" + linkText + "&lt;/a&gt;";
    document.getElementById("output").innerHTML = returnValue;
    }

    function formatRaw(url) {
    let returnValue = url;
    document.getElementById("output").innerHTML = returnValue;
    }

    function formatSelection() {
    let formatSelect;
    const radioButtons = document.querySelectorAll('input[name="formatStyle"]');
    for (const radioButton of radioButtons) {
        if (radioButton.checked) {
        formatSelect = radioButton.value;
        break;
        }
    }
    let url = document.getElementById('url').value;
    if (url == "") {
        return;
    }
    if (document.getElementById('strip-params').checked) {
        url = stripParams(url)
    }
    if ( formatSelect === "markdown"){
        formatMarkdown(url);
    }
    if ( formatSelect === "html"){
        formatHTML(url);
    }
    if ( formatSelect === "raw"){
        formatRaw(url);
    }
    }

    let urlForm = document.getElementById('url');
    urlForm.addEventListener("keyup", (event) => {
    if (event.code !== 'Enter' && event.code !== 'NumpadEnter')
    {
        return;
    }
    formatSelection();
    });

    let buttonSubmit = document.getElementById('btnSubmit');
    buttonSubmit.addEventListener("click", (event) => {
    formatSelection();
    });
{{< /js-inline >}}