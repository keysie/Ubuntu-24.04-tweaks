# Deploy Custom CA-certificates in Firefox for all Users

- use firefox-esr. just do.
- longer answer: there is some way to give firefox snap access to files like policies.json etc., but I'll look into that some other time
- create ```/etc/firefox/policies/policies.json```
- follow [this guide](https://mozilla.github.io/policy-templates/) for details, but basically use [this one](https://mozilla.github.io/policy-templates/#certificates--install)
- *do* specify a full path to that file - ```somehow using /usr/lib/mozilla/certificates``` does not work
