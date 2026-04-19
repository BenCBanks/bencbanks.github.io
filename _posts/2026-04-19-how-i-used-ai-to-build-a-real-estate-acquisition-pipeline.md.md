---
layout: post
title: How I Used AI to Build a Real Estate Acquisition Pipeline
type: Essay
description: How iterating with AI turned an idea — "find subdividable lots in Charlotte" — into a pipeline to identify investment candidates, including a ranked candidate list, and a process for off-market owner outreach.
tags: [meta]
---
Recently a creative and entrepreneurially minded friend of mine told me about how he was able to purchase a single family house on a lot, split it into 3 parcels — keeping the house as a rental and then build two new homes on the newly formed vacant lots. *"Genius, I want to try to do this too"* I thought — and there it started.

Before AI, capitalizing on an idea like this would require a few weekends, lots of hours on YouTube and message boards. We're all busy, especially our personal time after we're done working, so naturally the more time something takes, the less likely it gets done.

This post is about how AI leveled the playing field for me (and all of us) and how we should be using it to quickly capitalize on ideas and deliver value — it is now easier than ever to dream big and see it become a reality.

---

## The initial hypothesis

A bit of history: Charlotte's UDO — the Unified Development Ordinance that took effect June 1, 2023 — fundamentally changed what you can build in residential zones. Before it, 84% of Charlotte's residential land was zoned single-family only. After it, duplexes and triplexes became a buildable right in almost every zone across the city. In addition, zoning requirements changed and some areas saw minimum parcel size shrink. These changes created a massive shift in land value and opportunity.

My idea was simple (and inspired by my friend): there are parcels in Charlotte that look like ordinary single-family lots on the surface but are actually large enough to split, with the excess land now more valuable under the new rules. If I could find those parcels systematically — I had an edge.

Unfortunately, there are roughly 400,000 parcels in Mecklenburg County and I certainly could not manually review each one. I needed a way to filter them programmatically if this was going to have a chance to work.

---

## Building the parcel identification program

> "Do some deep analysis on the Charlotte GIS and Zoning regulations and the capability and feasibility to identify lots in populated areas which could be subdivided without changing the zoning laws for that area."

This was my first message to Claude — and afterwards one thing led to the next and we started moving pretty quickly. As it turns out, Mecklenburg County publishes excellent open data: parcel boundaries, ownership records, zoning classifications, sales history. The raw data was all there, but while I am pretty good at navigating relational data (e.g. SQL), any developer would laugh if I claimed that knowledge would help me sift through thousands of records in GeoJSON, XML, and Shape files.

I'd been tinkering with my Python skills off and on for a while — I built a basic scraper to pull the current tap list from a local brewery. With this familiarity, I decided to pursue creating a Python script to help analyze all the data I downloaded from Mecklenburg County.

### What the program does

The script loads six data sources:

- **Parcel shapefile** — zoning, lot size, geometry, acreage, etc.
- **Ownership GPKG** — assessed values and addresses
- **Street centerlines shapefile** — used to identify if a parcel is on a corner lot
- **Sales shapefile** — sale amount, date
- **Buildings shapefile** — used to identify the percentage of open land after accounting for building footprint
- **Flood zone shapefile** — identification if a parcel intersects with a flood zone

After loading the data, it joins them on parcel ID, computes geometry features, applies zone-specific filters, scores each candidate, and exports a ranked CSV with direct links to POLARIS (the county's property lookup) for each parcel.

The ranking was based on key features of each parcel using the following scoring system:

<style>
.score-grid{display:flex;flex-direction:column;gap:0;font-family:system-ui,sans-serif;margin:1.5rem 0}
.score-row{display:grid;grid-template-columns:160px 1fr;border-bottom:1px solid #e5e5e0}
.score-row:last-child{border-bottom:none}
.score-key{padding:14px 16px 14px 0;display:flex;flex-direction:column;gap:4px}
.score-name{font-size:13px;font-family:monospace;font-weight:600;color:#1a1a1a}
.score-pts{font-size:12px;color:#888}
.score-body{padding:14px 0;border-left:1px solid #e5e5e0;padding-left:16px}
.score-desc{font-size:13px;color:#555;line-height:1.5;margin:0 0 8px}
.tiers{display:flex;flex-wrap:wrap;gap:6px;margin-top:6px}
.tier{font-size:11px;padding:3px 8px;border-radius:4px;display:inline-flex;align-items:center;gap:4px;font-family:monospace}
.t-green{background:#eaf3de;color:#3b6d11}
.t-blue{background:#e6f1fb;color:#185fa5}
.t-gray{background:#f1efe8;color:#5f5e5a}
.t-red{background:#fcebeb;color:#a32d2d}
.total-row{margin-top:1rem;padding:14px 16px;background:#f8f8f6;border-radius:8px;display:flex;align-items:center;justify-content:space-between}
.total-label{font-size:13px;font-weight:600;color:#1a1a1a;font-family:monospace}
.total-sub{font-size:12px;color:#888;margin-top:2px}
.total-score{font-size:20px;font-weight:600;color:#1a1a1a}
</style>

<div class="score-grid">
  <div class="score-row">
    <div class="score-key"><span class="score-name">zone_score</span><span class="score-pts">1 – 5 pts</span></div>
    <div class="score-body">
      <p class="score-desc">Zone density — smaller minimum lot size means more lots per parcel and more value.</p>
      <div class="tiers"><span class="tier t-gray">N1-A → 1</span><span class="tier t-gray">N1-B → 2</span><span class="tier t-blue">N1-C → 3</span><span class="tier t-blue">N1-D → 4</span><span class="tier t-green">N1-E → 5</span><span class="tier t-green">N1-F → 5</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">sell_score</span><span class="score-pts">uncapped</span></div>
    <div class="score-body">
      <p class="score-desc">Total lots the parcel can yield. More lots is always better — no ceiling.</p>
      <div class="tiers"><span class="tier t-gray">lots_to_sell × 2</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">shape_bonus</span><span class="score-pts">0 – 1 pt</span></div>
    <div class="score-body">
      <p class="score-desc">Rectangular lots split more cleanly and are easier to develop.</p>
      <div class="tiers"><span class="tier t-green">shape_score ≥ 0.65 → +1</span><span class="tier t-gray">otherwise → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">corner_bonus</span><span class="score-pts">0 – 3 pts</span></div>
    <div class="score-body">
      <p class="score-desc">Corner lots have frontage on two streets, making it easier to give each new lot its own access.</p>
      <div class="tiers"><span class="tier t-green">corner → +3</span><span class="tier t-gray">unknown → +1</span><span class="tier t-gray">not corner → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">absentee_bonus</span><span class="score-pts">0 – 1 pt</span></div>
    <div class="score-body">
      <p class="score-desc">Absentee owners are more likely to be receptive to off-market offers.</p>
      <div class="tiers"><span class="tier t-green">absentee → +1</span><span class="tier t-gray">owner-occupied → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">proximity_bonus</span><span class="score-pts">0 – 3 pts</span></div>
    <div class="score-body">
      <p class="score-desc">Closeness to uptown Charlotte — nearer lots command higher resale value.</p>
      <div class="tiers"><span class="tier t-green">0–3 mi → +3</span><span class="tier t-blue">3–5 mi → +2</span><span class="tier t-gray">5–8 mi → +1</span><span class="tier t-gray">8+ mi → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">footprint_bonus</span><span class="score-pts">0 – 2 pts</span></div>
    <div class="score-body">
      <p class="score-desc">Small building footprint relative to lot size means more open land and cheaper redevelopment.</p>
      <div class="tiers"><span class="tier t-green">&lt;15% coverage → +2</span><span class="tier t-blue">15–30% → +1</span><span class="tier t-gray">≥30% → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">zip_price_bonus</span><span class="score-pts">0 – 8 pts</span></div>
    <div class="score-body">
      <p class="score-desc">High-value ZIP codes based on recent median sale prices. Fixed absolute tiers — a $500k ZIP always outscores a $300k ZIP regardless of dataset mix.</p>
      <div class="tiers"><span class="tier t-green">≥$600k → +8</span><span class="tier t-green">$500k–600k → +6</span><span class="tier t-blue">$400k–500k → +4</span><span class="tier t-gray">$300k–400k → +2</span><span class="tier t-gray">&lt;$300k → +0</span></div>
    </div>
  </div>
  <div class="score-row">
    <div class="score-key"><span class="score-name">flood_penalty</span><span class="score-pts">−3 pts</span></div>
    <div class="score-body">
      <p class="score-desc">Flood zone parcels face mandatory insurance, elevation requirements, and restricted building placement — all of which reduce value and marketability.</p>
      <div class="tiers"><span class="tier t-red">in flood zone → −3</span><span class="tier t-gray">not in flood zone → 0</span></div>
    </div>
  </div>
  <div class="total-row">
    <div><div class="total-label">priority score</div><div class="total-sub">Sum of all components. Sort descending to surface the best candidates first.</div></div>
    <div class="total-score">max ~19</div>
  </div>
</div>

---

## A critical data point

One geometry problem that took a while to solve was how to identify street frontage — a key data point when determining how many lots a parcel can be split into, since zoning rules set a minimum lot width based on street frontage.

Using spatial functions was new to me, but through trial and error I started to understand it more. Initially, I started by adding a 35-foot buffer to the centerline of the street to account for the right-of-way, and then determine how much of the street intersected with the parcel shape.

The problem with this approach was that it had a tendency to overestimate the frontage. The distance between the centerline of the street and the beginning of the parcel was not a constant 17.5 feet — right-of-ways are variable and neighborhood dependent. Therefore, the frontage measurement sometimes included part of the side length of the parcel, not only the front face.

<figure style="margin:1.75rem 0">
<svg width="100%" viewBox="0 0 680 400" role="img" xmlns="http://www.w3.org/2000/svg">
  <title>Frontage overestimation diagram</title>
  <desc>Two diagrams side by side. Left shows a consistent right-of-way where the buffer measures correctly. Right shows a variable right-of-way where the buffer bleeds onto the parcel sides, inflating the measurement.</desc>
  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
    <marker id="arr2" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="5" markerHeight="5" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>
  <style>.dg text{font-family:system-ui,sans-serif;fill:#1a1a1a}.dg .th{font-size:14px;font-weight:600}.dg .ts{font-size:12px;fill:#555}.dg .tr{font-size:12px;fill:#a32d2d}.dg .tg{font-size:12px;fill:#3b6d11}.dg .tb{font-size:12px;fill:#0c447c}</style>
  <g class="dg">
    <text class="th" x="160" y="22" text-anchor="middle">Consistent right-of-way</text>
    <text class="ts" x="160" y="38" text-anchor="middle">Buffer stops at parcel edge</text>
    <rect x="30" y="290" width="260" height="60" fill="#2C2C2A"/>
    <text x="160" y="327" text-anchor="middle" font-family="system-ui,sans-serif" font-size="14" font-weight="600" fill="#F1EFE8">Street</text>
    <line x1="30" y1="320" x2="290" y2="320" stroke="#F1EFE8" stroke-width="1" stroke-dasharray="8 5" opacity="0.5"/>
    <rect x="30" y="235" width="260" height="55" fill="#FAEEDA" opacity="0.85"/>
    <text class="ts" x="36" y="268" fill="#854F0B">35ft buffer</text>
    <rect x="80" y="80" width="160" height="155" rx="3" fill="#378ADD" opacity="0.85"/>
    <text x="160" y="163" text-anchor="middle" font-family="system-ui,sans-serif" font-size="14" font-weight="600" fill="#E6F1FB">Parcel</text>
    <line x1="80" y1="248" x2="240" y2="248" stroke="#0C447C" stroke-width="2" marker-start="url(#arr2)" marker-end="url(#arr2)"/>
    <text class="tb" x="160" y="262" text-anchor="middle">frontage = parcel width</text>
    <rect x="118" y="56" width="84" height="20" rx="10" fill="#eaf3de"/>
    <text class="tg" x="160" y="70" text-anchor="middle">correct</text>
    <text class="th" x="510" y="22" text-anchor="middle">Variable right-of-way</text>
    <text class="ts" x="510" y="38" text-anchor="middle">Buffer bleeds onto parcel sides</text>
    <rect x="380" y="290" width="260" height="60" fill="#2C2C2A"/>
    <text x="510" y="327" text-anchor="middle" font-family="system-ui,sans-serif" font-size="14" font-weight="600" fill="#F1EFE8">Street</text>
    <line x1="380" y1="320" x2="640" y2="320" stroke="#F1EFE8" stroke-width="1" stroke-dasharray="8 5" opacity="0.5"/>
    <rect x="380" y="235" width="260" height="55" fill="#FAEEDA" opacity="0.85"/>
    <text class="ts" x="386" y="268" fill="#854F0B">35ft buffer</text>
    <rect x="430" y="80" width="160" height="200" rx="3" fill="#378ADD" opacity="0.85"/>
    <text x="510" y="175" text-anchor="middle" font-family="system-ui,sans-serif" font-size="14" font-weight="600" fill="#E6F1FB">Parcel</text>
    <rect x="430" y="235" width="8" height="45" fill="#E24B4A" opacity="0.75"/>
    <rect x="582" y="235" width="8" height="45" fill="#E24B4A" opacity="0.75"/>
    <line x1="430" y1="310" x2="590" y2="310" stroke="#0C447C" stroke-width="2" marker-start="url(#arr2)" marker-end="url(#arr2)"/>
    <text class="tb" x="510" y="323" text-anchor="middle">true frontage</text>
    <line x1="430" y1="248" x2="590" y2="248" stroke="#E24B4A" stroke-width="2" marker-start="url(#arr2)" marker-end="url(#arr2)"/>
    <text class="tr" x="510" y="262" text-anchor="middle">measured (inflated)</text>
    <line x1="415" y1="210" x2="433" y2="250" stroke="#E24B4A" stroke-width="1" marker-end="url(#arr)"/>
    <text class="tr" x="393" y="204" text-anchor="middle">side edge</text>
    <text class="tr" x="393" y="216" text-anchor="middle">counted</text>
    <line x1="607" y1="210" x2="587" y2="250" stroke="#E24B4A" stroke-width="1" marker-end="url(#arr)"/>
    <text class="tr" x="629" y="204" text-anchor="middle">side edge</text>
    <text class="tr" x="629" y="216" text-anchor="middle">counted</text>
    <rect x="462" y="56" width="96" height="20" rx="10" fill="#fcebeb"/>
    <text class="tr" x="510" y="70" text-anchor="middle">overestimated</text>
    <line x1="340" y1="20" x2="340" y2="365" stroke="#D3D1C7" stroke-width="0.5"/>
    <rect x="30" y="370" width="12" height="12" rx="2" fill="#378ADD" opacity="0.85"/>
    <text class="ts" x="48" y="381">parcel</text>
    <rect x="110" y="370" width="12" height="12" rx="2" fill="#FAEEDA"/>
    <text class="ts" x="128" y="381" fill="#854F0B">35ft buffer</text>
    <rect x="230" y="370" width="12" height="12" rx="2" fill="#E24B4A" opacity="0.75"/>
    <text class="tr" x="248" y="381">erroneously included area</text>
  </g>
</svg>
<figcaption style="font-size:0.8rem;color:#888;text-align:center;margin-top:0.5rem">Left: a deep parcel where the buffer aligns cleanly with the front face. Right: when the right-of-way is narrower than expected, the buffer bleeds onto the sides — inflating the measured frontage.</figcaption>
</figure>

To help fix this, and after some brainstorming with my AI assistant, I calculated the smallest rectangle which the parcel can fit into and then determined the smallest side of that rectangle — the Minimum Rectangular Width (MRW).

Once both values were calculated, the MRW was used as a ceiling on the frontage measurement. If the street-intersection frontage came back higher than the MRW — which can happen on corner lots where the parcel boundary touches two streets — it was capped down to the MRW. If no frontage data was available at all, the MRW served as the fallback estimate. This ensured that the data was always constrained by a more realistic width, not an inflated one.

---

## What AI actually did in this process

The AI didn't have some magical insight I couldn't have found on my own. The UDO is public, the county data is public, and the geometry functions are documented. What it did was compress the development time dramatically — when I had a question about how minimum rotated rectangles work in GeoPandas, I got an explanation and code in the same conversation. When I misunderstood how sublots interact with lot standards, I got the specific article and footnote that clarified it.

The information was all there; AI just helped collapse the time between question and answer from hours or days into minutes.

---

## Where I am now

What the output can't tell me is condition, seller motivation, deed restrictions, or whether the math actually works for a specific deal. The real work starts after the script runs.

The deed research piece was something I underestimated at first. I found a property that looked great on every metric — right zone, right size, absentee owner who'd held it for 30 years. Then I pulled the covenant recorded in 1951 and read: *"No lot shall be subdivided into lots having less than 120 foot frontage... nor may any lot be subdivided to make its minimum size less than 20,000 square feet."* Usually deed restrictions override the UDO, and plenty of Charlotte's neighborhoods have them.

The initial output contained ~3,500 parcels which passed the critical filters (e.g. no flood zone) and were ranked by score. Slowly I have been going through each one from top to bottom, reviewing them on the POLARIS website to check their viability and assigning them a categorical ranking: A, B, C, or X for removal.

From the top ~300, I have identified 35 'A' parcels, 75 'B' parcels, and ~200 'C' or 'X' parcels.

For the top candidates — 'A' parcels — I will write a handwritten letter to their owners' mailing address, which is usually different from the physical address since greater weight was given to non-owner occupied parcels. Where possible, I will also try to contact them by phone. For all the 'B' and 'C' candidates, I will create a mail-merge CSV and print personalized letters explaining who I am and my interest in their property.

AI did not generate the idea for me — but it made the idea attainable and accessible in a way we've never had before. In many ways, the introduction of AI has leveled the playing field for all dreamers — an idea can be reality in just a few hours.

*Stay tuned to see if this project leads to an actual purchase…*