# Working 

Download a release matching what's showing in build.sh and extract hugo.
https://github.com/gohugoio/hugo/releases/tag/v0.160.1
`tar xvf hugo*.tar.gz hugo`

You may need to be able to run the hugo server from a github codespace. Here's the snippet to use `hugo server -D --appendPort=false --baseURL https://$CODESPACE_NAME-1313.$GITHUB_CODESPACES_PORT_FORWARDING_DOMAIN`

