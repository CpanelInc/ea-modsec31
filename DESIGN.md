# Mod Security 3 Support

## Target Audiences

1. Maintenance and security teams
2. Training and technical support
3. Managers and other internal key stakeholders
4. Future project/feature owners/maintainers

## Detailed Summary

We want EA4 to support mod security 3.x because:

1. 2.x is EOL (conceptually at least, the versions are very nebulously defined)
2. 3.x is easier to get into nginx

Some of the effort here (namely, naming) was done while [RPMifying OWASP rules](https://enterprise.cpanel.net/projects/EA4/repos/ea-modsec2-rules-owasp-crs/browse/DESIGN.md), specifically the “Prep for modsec3” section.

## Overall Intent

1. Make it easy for customers to get the latest rule sets and the latest mod security.
2. Have consistent behavior between nginx and Apache.
3. Give 3rd parties a consistent and universal way to provide rule sets for various mod sec versions.

### Piece 1 Intent

Since we RPMified mod sec 2.x’s OWASP 3.x the use of RPMs for rule vendors and the naming pattern of them is set.

See [RPMifying OWASP rules](https://enterprise.cpanel.net/projects/EA4/repos/ea-modsec2-rules-owasp-crs/browse/DESIGN.md), specifically the “Prep for modsec3” section, for details.

### Piece 2 Intent

Instead of trying to provide an EOL version of mod security for nginx (possibly but kludgy) it would be better to move to 3.x.

### Piece 3 Intent

This is pretty much the same as “Piece 1 Intent” except from the vendor’s prespective.

We will need a document for vendor’s that explain how to package rulesets for EA4 and WHM to consume.

## Maintainability

Estimate:

1. how much time and resources will be needed to maintain the feature in the future
2. how frequently maintenance will need to happen

The same as any other RPM. Automated updates via `ea4-tool`.

## Options/Decisions

### Requisite Info

[Felipe Zimmerle 3/2019 at Black Hat Asia](https://portswigger.net/daily-swig/waf-reloaded-modsecurity-3-1-showcased-at-black-hat-asia):

> In 3.0 the idea was to make it completely compatible with previous versions.
>
> Now, with 3.1, we are improving the performance and making it easier for people to extend ModSecurity and write their own rules.

1. ModSec 2.x is EOL
   1. it is an apache module, not a library
2. Modsec 3.0 is the latest stable: a kind of in between version to get to what they really want to, it is intended to be backwards compatible w/ 2.x
3. 3.1 (beta/alpha/???) is the end goal and has at least and behaves differently than 2.9 and 3.0
   1. `SecDefaultAction` behaves differently in 3.1, the last one found for a request is what is in effect. previously they got merged
   2. Some built-in limits are removed or changed to configuration of some sort
   3. greater malware detection capabilities through YARA support
   4. inject new rules in runtime (i.e. no webserver restart required)
   5. likely to be others
4. 3.x is [a library](https://github.com/SpiderLabs/ModSecurity) w/ “connectors” for individual web servers
   1. [nginx connector](https://github.com/SpiderLabs/ModSecurity-nginx) is stable
   2. [Apache connector](https://github.com/SpiderLabs/ModSecurity-apache) is alpha, expected to be [beta by end of 2020](https://github.com/SpiderLabs/ModSecurity-apache/milestones)
5. Felipe Zimmerle confirmed Aug 17, 2020 in our thread that “The rules will be the same for 2.9, 3.0, and 3.1.”

### Descisions

#### Rules

With the Aug 17, 2020 clarification we have more options:

| Approach | Pros | Cons | Notes |
|----------|------|------|-------|
| Keep original naming, seperate git repos | if they are different (our original understanding) pretty much has to be done this way | if they are the same, this will be a lot of duplication | N/A |
| Keep original naming, in git repo w/ `macros/` | no duplication, distinct package names | if they ever do deviate then this will need broken up into seperate git repos | N/A |
| Drop the version from the name | Very simple if they really are identical | If they ever do deviate we’ll need to revisit the versioned naming | e.g. `ea-modsec2-rules-owasp-crs` would become `ea-modsec-rules-owasp-crs` |

_Suggestion_: once we have 3.0 and/or 3.1 install `ea-modsec2-rules-owasp-crs` (it has no requires/conflicts) and see if they really do work. Then revisit this.

#### Config

ULC has “modsec2” hard coded everywhere and this is the current setup:

* /etc/apache2/conf.d/ea-modsec2.conf ➜ overall options for Apache
* /etc/apache2/conf.d/modsec/ea-modsec2.cpanel.conf ➜ vendor rules on/off
* /etc/apache2/conf.d/modsec/ea-modsec2.user.conf ➜ arbitrary custom rules

| Approach | Pros | Cons | Notes |
|----------|------|------|-------|
| Have ea-modsec3N use `modsec2` | no ULC changes | confusing |
| Have ea-modsec3N use `modsec3N`, make files with `modsec2` in their name be a symlink to their `modsec3N` counterpart | no ULC changes | less confusing |
| Have ea-modsec3N use `modsec3N`, update ULC to use the files for the installed version | the ideal thing since everything is seperate | a lot of work, that can’t be back ported |

e.g. of how 3.1 would look:
* ea-modsec31-connector-apache24 ➜
   * /etc/apache2/conf.d/ea-modsec31.conf ➜ overall options for Apache
   * /etc/apache2/conf.d/modsec/ea-modsec31.cpanel.conf ➜ vendor rules on/off
   * /etc/apache2/conf.d/modsec/ea-modsec31.user.conf ➜ arbitrary custom rules
* ea-modsec31-connector-nginx ➜
   * similar, ideally from same data source

Note: In order for both cnnectors to share the same rules, every rules RPM (using ea-modsec31-rules-owasp-crs as an example):

`/etc/apache2/conf.d/modsec_vendor_configs/OWASP3` will need:
1. its contents to be in `/opt/cpanel/ea-modsec2-rules-owasp-crs/OWASP3`
2. to be a hard link to ^^^ (if it is a symlink the WHM system does not pick up rules)

#### Mod Security Versions

No matter what:

1. We’d need ea-apache24-mod_security2 and each ea-modsecNN library to conflict with each other.
2. Since the apache connector is alpha mod sec 3.x needs to go in experimental (it is OK since `ea-nginx` is experimental too).

| Approach | Pros | Cons | Notes |
|----------|------|------|-------|
| Do 3.0 and 3.1 | We’d be able to assess both (and hopefully sort out some of the confusion) | more effort | Probably should do this so we have max info/options.  |
| Do 3.0 | currently stable and is supposed to have the same behavior as 2.9 | its a tad confusing how they refer to “stable”, its still sort of being worked on and 2.9 is the stable one, but 2.9 EOL in the sense that they want to work on 3.x instead of 2.9 | we could start w/ this and do 3.1 later |
| Do 3.1 | skip to the new hotness | behavior change may be confusing, still being worked on? | This was the original tack with the info we had but we should probably do one of the 3.0 approaches since it turns out to be “stable”. |
| In addition to ^^^ also do 2 for nginx | nginx would do have all versions like apache | it is EOL, may not actually be possible, if it is what we have found looks pretty kludgy | Hard no, lol |

### Initial Approach

* **Rules**: do the _Suggestion_ to better be able to see what we could/should do
* **Config**: do the symlink for now and revisit fixing ULC later
* **Mod Security Versions**: do 3.0 and 3.1 so we can get solid answers to what works/behaves with what
   * **Update**: only 3.0 for now, see ZC-7374 description for details

**Note**: _We are not going to support modsec 3 on CentOS 6 at this time._ Because it is becoming increasingly difficult to get new things to build on C6 and since we only support C6 on the current LTS. It is another incentive for people to update.

## Child Documents

None.

