+++
date = '2026-07-01T11:01:58-06:00'
draft = false
title = 'Helm Repo Parse'
+++
{{< form-field >}}

{{< helm-table  >}}

{{< js-import src="https://cdnjs.cloudflare.com/ajax/libs/js-yaml/4.1.0/js-yaml.min.js">}}

{{< output-div >}}
{{< js-inline >}}
    let urlForm = document.getElementById('url');
    urlForm.addEventListener("keyup", (event) => {
    if (event.code !== 'Enter' && event.code !== 'NumpadEnter')
    {
        return;
    }
    parseYaml();
    });

    let buttonSubmit = document.getElementById('btnSubmit');
    buttonSubmit.addEventListener("click", (event) => {
    parseYaml();
    });
    function parseYaml() {
        const urlInput = document.getElementById('url');
        let urlValue = urlInput.value.trim();

        if (urlValue === "") {
            return;
        }

        // Logic: Append /index.yaml if it's missing
        if (!urlValue.endsWith('/index.yaml')) {
            // Ensure there's a slash between the base URL and index.yaml
            urlValue = urlValue.replace(/\/$/, "") + '/index.yaml';
        }

        fetch(urlValue)
            .then(r => {
                if (!r.ok) throw new Error("Network response was not ok");
                return r.text();
            })
            .then(r => {
                const yaml = jsyaml.load(r);
                const tableBody = document.querySelector('#helmTable tbody');
                
                // Clear existing rows before adding new ones
                tableBody.innerHTML = "";

                for (let entry in yaml['entries']) {
                    // Helm index files usually store the latest version at index 0
                    let temp = yaml['entries'][entry][0];
                    let row = tableBody.insertRow();
                    row.insertCell().textContent = temp.name || '';
                    row.insertCell().textContent = temp.version || '';
                    row.insertCell().textContent = temp.appVersion || '';
                    row.insertCell().textContent = temp.description || '';
                }
            })
            .catch(err => {
                console.error(err);
                alert("Failed to fetch or parse the YAML file. Check the console for details.");
            });
    }
{{< /js-inline >}}
