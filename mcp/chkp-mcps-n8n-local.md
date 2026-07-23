# Local N8N with Check Point MCPs

## Prerequisites

```shell
# node LTS installed
node -v # v24 is current LTS

# npm installed
npm -v # something like 11.x.y

# IF NOT INSTALLED, fix using https://nodejs.org/en/download
```

## Start first Check Point MCP with HTTP transport

```shell
# based on https://npmx.dev/package/@chkp/management-logs-mcp#user-content-http-transport

# some work folder
cd $(mktemp -d)

# start - see arguments in https://npmx.dev/package/@chkp/management-logs-mcp#user-content-on-prem-management-example
bash -c 'MANAGEMENT_HOST=cpman.duckdns.org USERNAME=admin PASSWORD=demo123 npx @chkp/management-logs-mcp --transport http --transport-port 18601'
# now available on http://localhost:18601/mcp
```

## Minimal effort N8N new instance on localhost:5678

```shell
# some work folder
cd $(mktemp -d)
# start on http://localhost:5678 - more on https://npmx.dev/package/n8n
npx -y n8n
```

## Basic N8N Check Point Logs AI Agent

Consider importing workflow as a starter:

