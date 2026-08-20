# Shodan CLI for Windows

Copyright (C) Paolo Monti 2026. 

Command-line client for Shodan.

## Authentication

To store the API key persistently in Windows Credential Manager:

```powershell
.\shodan-cli.exe --save-api-key 'shodan-api-key'
```

The key is encrypted and managed by Windows for the current user. It does not
need to be passed to subsequent commands. To remove it:

```powershell
.\shodan-cli.exe --forget-api-key
```

For non-persistent use, set the environment variable:

```powershell
$env:SHODAN_API_KEY = 'shodan-api-key'
.\shodan-cli.exe info
```

Alternatively, use `--api-key <key>` for the current invocation only. Keep in
mind that command-line arguments may be visible to other local processes.

## Examples

```powershell
.\shodan-cli.exe host 8.8.8.8
.\shodan-cli.exe search product:nginx country:IT --page 1
.\shodan-cli.exe count port:22 --facets country,org
.\shodan-cli.exe count "port:22 country:IT"
.\shodan-cli.exe stats ssh
.\shodan-cli.exe stats ssh --facets country,org --limit 5
.\shodan-cli.exe myip
.\shodan-cli.exe count ssh --facets country --country IT
.\shodan-cli.exe dns-resolve example.com,openai.com
.\shodan-cli.exe domain example.com --type A
.\shodan-cli.exe exploit-search CVE-2024-3094
```

## Shodan preset catalog

The client starts with 1,817 presets inspired by Shodan Eye collections and
organized into the `web`, `database`, `remote-access`, `iot`,
`network-service`, `infrastructure`, `cloud`, `industrial`, and `security`
categories, plus the dedicated `cve` category. This category contains all
1,671 vulnerabilities in version `2026.08.19` of the CISA Known Exploited
Vulnerabilities catalog. On first use,
these presets seed an external database at
`%LOCALAPPDATA%\Shodan CLI\presets.db`. The file is not plaintext: its contents
are protected with Windows DPAPI for the current Windows user and are replaced
atomically after each change. The catalog can be browsed without an API key:

```powershell
.\shodan-cli.exe presets
.\shodan-cli.exe presets database
.\shodan-cli.exe presets --category iot
.\shodan-cli.exe presets web --json
```

`dorks` is an alias for `presets`. Each entry reports its name, category,
description, and complete Shodan query in the usual `field: value` format.

Presets can be added, edited, and deleted directly from the command line:

```powershell
.\shodan-cli.exe preset-add custom-ssh --category custom --description "SSH services" --query "port:22"
.\shodan-cli.exe preset-edit custom-ssh --description "SSH services in Italy" --query "port:22 country:IT"
.\shodan-cli.exe preset-delete custom-ssh --yes
.\shodan-cli.exe preset-db-info
```

`preset-reset --yes` replaces the current contents with the original 1,817 seed
presets. Destructive commands require `--yes`. The corresponding `dork-add`,
`dork-edit`, `dork-delete`, `dork-reset`, and `dork-db-info` aliases are also
available.

Each CVE preset uses the CVE identifier as its name and a Shodan `vuln:`
filter as its query:

```powershell
.\shodan-cli.exe presets --category cve
.\shodan-cli.exe preset cve-2021-44228 --action query
.\shodan-cli.exe preset cve-2021-44228 --action count --country IT
.\shodan-cli.exe preset eternalblue --action query
```

For example, `eternalblue` is the readable alias for `CVE-2017-0144` and generates the
query `vuln:CVE-2017-0144`. The standard `cve-2017-0144` preset remains
available as well.

CVE queries depend on Shodan indexing and on the account plan permitting the
`vuln` filter. Shodan may infer some vulnerabilities from detected software
versions, so an unverified match is a starting point for investigation rather
than proof that the host remains vulnerable.

Use `--preset-db FILE` with any preset command to select another database. A
DPAPI-protected database can be read only by the Windows user that created it
on the same Windows installation; copying it to another account does not expose
its contents and does not make it portable.

By default, `preset` performs a count without downloading individual hosts:

```powershell
.\shodan-cli.exe preset mongodb
.\shodan-cli.exe preset ssh --country IT
```

The `--action` option selects the desired behavior:

```powershell
.\shodan-cli.exe preset webcam --action query
.\shodan-cli.exe preset nginx --action stats --country IT --limit 5
.\shodan-cli.exe preset nginx --action search --country IT --page 1
.\shodan-cli.exe preset nginx --action recon --country IT --limit 100 --format csv --output nginx-it.csv
```

The available actions are `query`, `count`, `stats`, `search`, and `recon`.
`query` only displays the composed query and works offline; `search` and
`recon` are subject to Shodan plan requirements and query credits. `dork` is
an alias for `preset`.

Additional terms or filters can be placed after the preset name. The
`--country` filter is validated and added automatically:

```powershell
.\shodan-cli.exe preset apache port:443 --country IT
```

Presets identify technologies and services; their presence does not imply that
a system is vulnerable. Searches must only be used on assets you own or are
authorized to assess.

## Multi-page reconnaissance and export

The `recon` command adds the collection workflow commonly associated with
Shodan Eye while retaining the official APIs and the client's modular
structure. It automatically retrieves successive pages up to the requested
limit:

```powershell
.\shodan-cli.exe recon "product:nginx country:IT" --limit 100
```

The default limit is 100 results. `--all` removes the limit and must be
specified explicitly; it can perform many requests and consume query credits.
`--start-page N` resumes collection from a specific page. The search endpoint
requires a Shodan plan that permits result queries; the free plan may allow
`count` while rejecting `search` and `recon`.

The default format is a compact `field: value` profile. The initial fields are
IP address, port, organization, country, city, transport, domains, hostnames,
and banner. They can be replaced with `--fields`, which also supports dotted
JSON paths:

```powershell
.\shodan-cli.exe recon ssh --limit 50 --fields ip_str,port,org,location.city
```

Data can be exported as text, JSON Lines, or CSV:

```powershell
.\shodan-cli.exe recon ssh --limit 100 --output results.txt
.\shodan-cli.exe recon ssh --limit 100 --format jsonl --output results.jsonl
.\shodan-cli.exe recon ssh --limit 100 --format csv --output results.csv
```

If `--format` is omitted, the `.csv`, `.jsonl`, and `.ndjson` extensions select
the format automatically. In JSONL format, each line contains one raw JSON
match; `--json` is an alias for this format with the `recon` command.

An existing file is not overwritten. Use `--append` to add results or `--force`
to replace it. The two options cannot be combined. Progress is written to
standard error so that output intended for a pipe or file remains clean; it
can be disabled with `--no-progress`.

Interactive mode prompts for the query, limit, destination, and format when
they are not already present on the command line:

```powershell
.\shodan-cli.exe recon --interactive
```

For `search` and `count`, the `--country` option accepts a two-letter ISO
country code and generates the Shodan filter `country:CODE`. When the `country`
facet is requested, the previously supported abbreviated form is also
available:

```powershell
.\shodan-cli.exe count ssh --facets country it
```

In this case, `it` is interpreted as `country:IT`, not as free text.

On `cmd.exe`, queries containing spaces should be enclosed in double quotes.
For compatibility, the client also accepts surrounding single quotes and
normalizes the common `filter::value` mistake to `filter:value`. Therefore,
`count 'port::22 country:IT'` is interpreted as
`count "port:22 country:IT"`.

The normal `count` output shows `total` and any facets without the empty
`matches` field. The `--json` option preserves the original JSON response.

By default, `stats` uses the `country,org` facets and returns up to 10 values
for each facet. The number can be changed with `--limit`, while `--facets`
selects the desired facets.

The `search`, `count`, `stats`, and `recon` commands accept `dork NAME` or
`preset NAME` in place of a literal query. The preset is expanded before the
request, so this command sends Drupal's real query to Shodan:

```powershell
.\shodan-cli.exe stats dork drupal
```

The effective query sent to the API is always displayed as `query: value`.
For normal text output it is written to standard output. When stdout contains
raw JSON, JSONL, or CSV data, the query is written to standard error so the
machine-readable output remains valid.

The `myip` command shows the public IP address detected by Shodan in the `ip`
field. Use `myip --ipv6` to attempt the request through the IPv6 endpoint.

Active scans are blocked by default. They require the explicit
`--allow-active-scan` option and a compatible Shodan plan:

```powershell
.\shodan-cli.exe scan-submit 192.0.2.1 --allow-active-scan
```

Use `--help` for the complete command list.

By default, results are displayed as `field: value`. Shodan facet arrays
containing `value` and `count` are represented directly as `value: count`, for
example:

```text
facets.country: 2
  CN: 17237256
  US: 5711053
```

Add `--json` to obtain the raw JSON response:

```powershell
.\shodan-cli.exe host 8.8.8.8 --json
```

Errors are written exclusively to standard error in the following format:

```text
Error: message
```

Unknown command-line options are rejected. For example, `--categoy` produces
`Error: Unknown option: --categoy.` instead of being silently ignored.
Unknown commands are handled in the same way, before API-key validation; for
example, `serach` produces `Error: Unknown command: serach.`.
