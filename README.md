# Arch Linux Security Tracker Tools

This is a collection of Python scripts to make working with the [Arch Linux
Security Tracker](https://github.com/archlinux/arch-security-tracker) easier.

## Features

* CVE entry parsing from multiple sources (currently
  [NVD](https://nvd.nist.gov/),
  [Mozilla](https://www.mozilla.org/en-US/security/advisories/),
  [Chromium](https://chromereleases.googleblog.com/) and
  [GitLab](https://gitlab.com/gitlab-org/cves)) into a JSON format consumable
  by the tracker
* Automatic batch addition of the parsed CVE entries to the tracker

## Dependencies

* python >= 3.6
* python-lxml

## CVE entry parsing

CVEs from multiple sources can be parsed. All parser scripts take the CVEs to
be considered as a list of arguments and write the parsed CVE entries to stdout
in JSON form. The JSON format follows the one used by the tracker as part of
its API endpoints, e.g. <https://security.archlinux.org/CVE-2019-9956.json>.

### NVD

[`tracker_get_nvd.py`](tracker_get_nvd.py) parses CVE entries from the official [NVD
database](https://nvd.nist.gov/). It is used as

```sh
./tracker_get_nvd.py CVE...
```

Description and references are taken verbatim from the NVD CVE entry. Severity
and attack vector are derived from the CVSS v3 if present (this usually takes a
few day after the CVE has been published). The type of the vulnerability is
always set to "Unknown" and needs to be filled by hand by the user.

This is mostly included as an example for working with the JSON format. CVEs
obtained from this source often require manual changes to the description and
references before they can be used for the tracker.

### Mozilla

[`tracker_get_mozilla.py`](tracker_get_mozilla.py) parses CVEs issued by
[Mozilla](https://www.mozilla.org/en-US/security/advisories/), mostly for
Firefox and Thunderbird. It is used as

```sh
./tracker_get_nvd.py CVE... MFSA...
```

where `MFSA` is an advisory number issued by Mozilla, e.g.
[`mfsa2021-01`](https://www.mozilla.org/en-US/security/advisories/mfsa2021-01/).
If a MFSA is specified, all CVEs included in this advisory will be parsed.

Description, references and severity are taken verbatim from the Mozilla
advisory. The attack vector is assumed to be "Remote" by default due to the
nature of the Mozilla products. The type of the vulnerability is always set to
"Unknown" and needs to be filled by hand by the user.

### Chromium

[`tracker_get_chromium.py`](tracker_get_chromium.py) parses CVEs issued for
[Chrome](https://chromereleases.googleblog.com/). It is used as

```sh
./tracker_get_chromium.py URL...
```

where `URL` is the URL of a Chrome release blog post, e.g.
<https://chromereleases.googleblog.com/2021/05/stable-channel-update-for-desktop.html>.

The description is of the form "A `type` security issue has been found in the
`component` component of the Chromium browser before version `new_version`.",
where `type`, `component` and `new_version` are parsed from the blog post. The
corresponding severity is taken from the blog post as well. The URL of the blog
post and the link to the corresponding Chromium bug report as specified in the
blog post are used as references. The attack vector is assumed to be "Remote"
by default as Chromium is a browser. The type of the vulnerability is always
set to "Unknown" and needs to be filled by hand by the user.

### GitLab

[`tracker_get_gitlab.py`](tracker_get_gitlab.py) parses CVE entries assigned by
the [GitLab CNA](https://gitlab.com/gitlab-org/cves), for the GitLab products
as well as some projects hosted on GitLab. These CVEs are usually added to the
NVD database quite quickly as well, but the GitLab entries have more detailed
information regarding the CVSS score quicker. The parser is used as

```sh
./tracker_get_gitlab.py CVE...
```

Description and references are taken verbatim from the NVD CVE entry. Severity
and attack vector are derived from the CVSS v3. The type of the vulnerability
is always set to "Unknown" and needs to be filled by hand by the user.

## CVE upload to the security tracker

[`tracker_add.py`](tracker_add.py) adds CVEs to the Arch Linux Security
Tracker. It reads a JSON file generated by one of the parsers from stdin and
tries to create a new CVE for each of the items found in there. The necessary
login credentials can be supplied using the `TRACKER_USERNAME` and
`TRACKER_PASSWORD` environment variables, or will otherwise be asked queried on
the TTY. 

Note that only adding new CVEs is supported at the moment. Trying to add an
already existing CVE will try to merge the data according to the upstream
tracker logic, which will only partially succeed if the data is conflicting.

The URL to the tracker is set as <https://security.archlinux.org> by default,
but can be changed for debugging purposes by setting the `TRACKER_URL`
environment variable, e.g. to a tracker instance running locally:

```sh
TRACKER_URL='http://127.0.0.32:5000' ./tracker_add.py
```

## Example workflow

1. Download a set of CVEs using one of the parsers to a JSON file, e.g.

    ```sh
    ./tracker_get_mozilla.py mfsa2021-01 > mfsa2021-01.json
    ```

2. Edit the file to check the generated data and add missing information like
the vulnerability type:

    ```sh
    $EDITOR mfsa2021-01.json
    ```

3. Upload the CVEs to the tracker:

    ```sh
    ./tracker_add.py < mfsa2021-01.json
    ```

If you are feeling brave, you can omit the editing step and directly upload the
generated data to the tracker:

```sh
./tracker_get_mozilla.py mfsa2021-01 | ./tracker_add.py
```

Missing or incorrect information can be edited afterwards using the web
interface of the tracker. Be careful with this approach, mass-editing messed up
CVE entries has not been implemented yet...

## TODO

* Implement more parsers
* Validation of the generated JSON files, at least in `tracker_add.py`
* Better error handling
* [SSO support using Keycloak](https://github.com/archlinux/arch-security-tracker/pull/181)
* Batch editing of existing CVEs
