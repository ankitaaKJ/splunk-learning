# Splunk: combined notes
## 1. Architecture and data pipeline

![Splunk architecture](images/arch.png)

**Forwarders** collect data on the source machines. A **universal forwarder (UF)** is a small agent that does minimal processing. A **heavy forwarder (HF)** is a full Splunk instance that can parse, filter, mask and route before forwarding. Both tag data with `host`, `source` and `sourcetype` before sending. **[checked]**

**Indexers** turn the stream into searchable events and store them in **buckets**: compressed raw data plus `.tsidx` index files. In a cluster, indexers replicate each other's buckets so losing one does not lose data. In a distributed setup they also act as **search peers**.

**Search heads** serve Splunk Web, the REST API and the CLI. They send the search to every indexer in parallel and merge the results. They also hold your **knowledge objects** (reports, dashboards, event types, extractions).

**Deployment server** distributes apps and configuration to forwarders (deployment clients). **[course]**

**Why this design.** Separating collection, storage and search lets each scale on its own. Thousands of endpoints get cheap forwarders. Storage grows by adding indexers. Analysts get more concurrency by adding search heads. In your single-VM lab all three roles live on one instance.

![Data pipeline](images/pipeline.png)

| Stage | What happens | Who can do it **[checked]** |
|---|---|---|
| Input | Read files, ports, scripts; split into 64 KB blocks; tag host, source, sourcetype, index | Indexer, UF, HF |
| Parsing | Break into events, set timestamps, merge multi-line events, apply regex rules | Indexer or HF (UF only for structured data) |
| Indexing | Write events, compressed raw data and `.tsidx` to buckets | Indexer only |
| Search | Run SPL, apply search-time field extractions, power reports, dashboards, alerts | Indexer or search head |

| | Universal forwarder | Heavy forwarder |
|---|---|---|
| Processing at source | Minimal | Full parsing, filtering, routing |
| Footprint | Small; suits thousands of endpoints | Full Splunk Enterprise install |
| Indexer workload | Parses and indexes | Indexes only (data already parsed) |
| Network | May carry noise you did not filter | Filtered at source, saves bandwidth |

## 2. Splunk Web, apps and add-ons (Lesson 2)

**Layout.** The Splunk bar holds the app switcher, Messages, Settings, Activity (jobs, triggered alerts) and Help. The home page lists installed apps on the left. Search & Reporting is where you write SPL, set time ranges, inspect fields and save reports, dashboards and alerts. **[course]**

**Apps vs add-ons.** An **app** has a user interface, dashboards and workflows. A **technology add-on (TA)** works behind the scenes: data collection, parsing, field extraction, and mapping to the Common Information Model (CIM). Rule of thumb: apps are for people, add-ons are for data. **[course]**

**App context matters.** Saved searches, dashboards and knowledge objects belong to the app that is active when you create them. If an object "disappeared", you are probably in a different app. Check the app switcher first.

**Your Qualys link.** The Splunkbase app you cited (Qualys Vulnerability Management, app 6565) is a **Splunk SOAR connector**, not something that ingests Qualys data into Splunk Enterprise. Its five actions are test connectivity, list asset groups, list host findings, launch scan and scan summary. **[course]** Separate from that, a Qualys technology add-on exists for pulling Qualys data into Splunk Enterprise **[verify]**. For interviews, the distinction is the point: SOAR connector = automate actions; TA = ingest data. Be aware you need Qualys API access for either, and your licence is gone, so treat this as a concept topic.

**Jobs and roles.** Activity > Jobs lets you inspect, pause or extend search jobs. What you see under Settings depends on your role.

---

## 3. Search fundamentals (Lesson 3) and search modes

### Terms and Boolean logic
Combine keywords, quoted phrases and wildcards. **Operator order is NOT, then OR, then AND**, with AND implied between terms. **[course; I did not re-check it in Splunk's docs]**

> Example: `Failed OR Accepted password` is read as `(Failed OR Accepted) AND password`. OR binds tighter than AND here, the reverse of many programming languages. Use parentheses whenever it matters.

### Fields
Default fields: `host`, `source`, `sourcetype`, `index`, `_time`. Filter with `=`, `!=`, `IN (...)` or numeric comparisons.

### Pipes and transforming commands
The pipe passes output left to right. Core commands: `fields`, `dedup`, `sort`, `table`, `stats` (count, dc, sum, values) and `timechart`. A *transforming* command turns events into a table or chart, which is why modes treat searches ending in `stats` differently.

### Search modes

![Search modes](images/modes.png)

| Mode | Field discovery | Use it for |
|---|---|---|
| Fast | Only fields named in the search | Known fields, searches ending in `stats`, `timechart`, `chart`, `top`, `rare` |
| Smart (default) | Adapts: Fast-like with a transforming command, Verbose-like for plain event searches | Everyday triage |
| Verbose | All discoverable fields, even with a transforming command | Unfamiliar sourcetypes; slowest on large ranges |

### Good habits
1. Tight time range first (last 15 minutes, not All time).
2. Filter with `index`, `sourcetype` and specific terms before any transforming command.
3. Summarise with `stats` instead of reading raw events.

### Practice with Splunk's tutorial data
Your dashboard doc ("BCG Logins") uses Splunk's **Search Tutorial data**: fictional Buttercup Games with `access.log`, `secure.log` and `vendor_sales.log`. The usernames in your chart (`djohnson`, `nsharpe`, `myuan`) match `secure.log`. Download link: on the Splunk Search Tutorial page "Upload the tutorial data". Upload it to your own lab so every example below runs on your data. **[checked: file list and sample event; the zip link is on the tutorial's next page]**

A `secure.log` failed login looks like this: `mailsv1 sshd[5276]: Failed password for invalid user appserver from 194.8.74.23 port 3351`. **[checked]** The examples use `rex` to extract fields, so they do not depend on how your upload named the sourcetype. Set the index to wherever you uploaded (the default is `main`). **[verify field names on your upload]**

**Example 1: failed logins by user (the pie chart in your notes)**

```
index=main source="*secure.log" "Failed password"
| rex "Failed password for (invalid user )?(?<user>\S+) from (?<src_ip>\d{1,3}(?:\.\d{1,3}){3})"
| stats count by user
| sort - count
| head 10
```
*What it proves:* which accounts are targeted. `root`, `admin` and `administrator` at the top is the classic signature of automated brute force against default names.

**Example 2: failures vs successes over time (the line chart)**

```
index=main source="*secure.log" ("Failed password" OR "Accepted password")
| eval outcome=if(searchmatch("Failed password"),"failure","success")
| timechart span=1h count by outcome
```
*What it proves:* a spike in failures followed by one success is the pattern that matters: possible brute force that worked.

**Example 3: failures by country (the map)**

```
index=main source="*secure.log" "Failed password"
| rex "from (?<src_ip>\d{1,3}(?:\.\d{1,3}){3})"
| iplocation src_ip
| stats count by Country
| sort - count
```
Replace the last two lines with `| geostats count` to feed a map panel. `iplocation` uses a GeoIP database that ships with Splunk. **[verify]**

---

## 4. Knowledge objects

Knowledge objects are **search-time** definitions that classify, normalise, enrich and interpret your data **without changing the raw events at index time**. This is the main reason you can fix a bad extraction later without re-ingesting. **[course]**

| Role in your diagram | Object | Example on the tutorial data |
|---|---|---|
| Classification | Event type, tag | Event type `ssh_fail` = `source="*secure.log" "Failed password"`, tagged `auth_failure` |
| Normalisation | Field alias, CIM mapping | Alias `clientip` and `c_ip` to `src_ip` so web and firewall logs correlate |
| Enrichment | Lookup, calculated field | Lookup `src_ip` to asset owner, criticality or geography |
| Interpretation | Field extraction, workflow action | A regex that extracts `user` and `src_ip` from `secure.log` |
| Structure | Data model | Authentication dataset that powers pivots and accelerated reports |

Your list omits **saved searches, macros and workflow actions**, which are also knowledge objects. A macro is a reusable chunk of SPL, which is useful for a detection you run in many searches. **[verify]**

**Lookup example.** Upload `asset_owners.csv` with columns `src_ip,owner,criticality`, then:

```
... | lookup asset_owners.csv src_ip OUTPUT owner criticality
```
*Why use a lookup:* the reference list changes on its own schedule. You update the CSV; no search changes. This is the right tool for allow-lists and asset inventories.

**Where they live.** Knowledge objects are stored with an app and have permissions (section 6), which is why app context matters.

---

## 5. Reports, dashboards and drilldown

### Reports
A report is a saved search that runs on demand or on a schedule and can feed a dashboard. To create one: run the search, then Save As > Report, enter a title and optional description, choose whether to include a time range picker, and save. Then pick Continue Editing, Add to Dashboard, Permissions or View. **[course]**

### Permissions
New reports are **Private (Owner)**. Under Permissions you choose who sees them:

| Display for | Meaning |
|---|---|
| Owner | Only you (default) |
| App | Users with access to the app |
| All apps | Everywhere in the deployment |

Then set **Read** (view and run) and **Write** (edit, change permissions, delete) per role.

**Example.** An analyst saves "Failed SSH Logins by Source IP". It starts Private. Set App-level sharing with Read for `user` and Write for `power` and `admin`, so every analyst can run it but only senior roles can change it. An admin can see and manage it even before you share it. **[course]**

### Dashboards
A dashboard is a set of panels (charts, tables, single values) driven by searches or reports.

| | Classic (Simple XML) | Dashboard Studio |
|---|---|---|
| Layout | Grid of rows and panels | Grid or Absolute (free-form canvas, images, shapes, layers) |
| Definition | XML | JSON |
| Best for | Quick operational views | Polished, interactive views |

### Dashboard Studio example: your "BCG Logins" dashboard
- **Failed logins by user** (pie): Example 1, `stats count by user`.
- **Failed logins by country** (map): Example 3 with `geostats`.
- **Failures vs successes over time** (line): Example 2.

For the plan's Day 17 Endpoint Overview dashboard, apply the same method with Sysmon data, and for each panel write down the decision it supports. A panel with no decision should go.

### Drilldown
Drilldown makes a click do something. The four actions: **link to a search** (opens a search filtered by the clicked value), **link to a dashboard** (passes tokens), **link to a URL**, and **set tokens** (filters other panels on the same dashboard). **[course]**

*Example:* a failed-logins-by-`src_ip` table; clicking an IP opens a search filtered to that IP. The clicked value arrives as a **token**. Token names depend on the dashboard type (for example `$click.value$` in Simple XML), so check the syntax for your version. **[verify]**

---

## 6. RBAC: roles and capabilities (corrected)

![RBAC](images/rbac.png)

**Core idea.** A **role** bundles capabilities, allowed and default indexes, search filters and search job limits. Users receive roles. **[course]**

| Role | What it does **[checked unless noted]** |
|---|---|
| `user` | Create, edit, run and save your own searches and preferences |
| `power` | Edit all shared objects (saved searches, alerts), tag events |
| `admin` | Manages most users, objects and configuration; the most capabilities |
| `can_delete` | Grants `delete_by_keyword`, required for the `delete` command **[checked]** |
| `splunk-system-role` | Internal tasks **[course]** |

**`delete` does not erase data.** It hides matching events from search results; the raw data stays in the buckets. **[course]**

**What happens with several roles (this is the correction):**
- **Capabilities:** the user gets the *most permissive* combination. **[checked]**
- **Allowed indexes:** the *most permissive* access across roles. **[checked]**
- **Search filters:** the filters are *combined* and the restrictions of each apply. **[checked]**
- **Search quotas:** Splunk's page does not say how these combine, so do not claim a rule.

*Why it matters:* roles only ever add access. You cannot take access away by giving someone a second, more restricted role. To restrict, remove the broader role or use a search filter. This is the answer to "someone has both `power` and a restricted analyst role: what can they search?"

---

## 7. Glossary

| Term | Meaning |
|---|---|
| Event | One record in an index; has `_time` and `_raw` |
| Index | Storage unit for events; set retention and access per index |
| `source` | Where the event came from (file path or port) |
| `sourcetype` | The event's format; decides line breaking and extraction |
| `host` | The machine that produced it |
| Bucket | On-disk directory holding raw data and `.tsidx` files |
| `.tsidx` | Index file that makes searches fast |
| Forwarder (UF/HF) | Agent that sends data to indexers |
| Indexer | Parses (if needed), stores and searches data |
| Search head | The component users search from |
| Deployment server | Pushes apps and config to forwarders |
| SPL | Search Processing Language |
| Transforming command | Turns events into tables or stats (`stats`, `timechart`, `top`) |
| Knowledge object | Search-time definition: extraction, alias, lookup, event type, tag, macro, data model |
| Lookup | CSV or similar joined to events at search time |
| CIM | Common Information Model; standard field names across sources |
| TA / add-on | Collection, parsing and normalisation package |
| App | Package with UI, dashboards and workflows |
| Token | Variable in a dashboard filled by inputs or clicks |
| Drilldown | Click behaviour on a panel |
| Role / capability | Bundle of permissions / one permission |
| Search mode | Fast, Smart or Verbose; controls field discovery |

---

## 8. Hands-on tasks for your lab 

| Plan day | Task | Proof to save |
|---|---|---|
| 1 | Licensing screenshot; snapshot `splunk-clean` | Screenshot and snapshot name |
| 3 | Forward Sysmon and Security logs; `stats count by EventCode` | Screenshot |
| 5-6 | Upload the tutorial data; run Examples 1-3 in Fast, Smart and Verbose and note field differences | Three screenshots and one sentence each |
| 15 | Ten searches, each with the question it answers | `spl/searches.md` |
| 16 | Event type + tag, one lookup, one macro | Screenshots |
| 17 | Endpoint Overview dashboard with one drilldown | Screenshot and JSON |
| 18 | THM "Splunk: Exploring SPL" | Five commands and the question each answered |

---
