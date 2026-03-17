# BMS UI Patterns Reference

Guidance for handling common BMS web interface layouts during extraction.

> **Safety: Read-only.** All BMS interactions must be read-only. Avoid clicking write, override, command, or acknowledge buttons — these are common in BMS UIs and can modify real equipment. Only use navigation and tree-expansion clicks.

---

## Niagara N4 (Tridium / Fox)

**Login:** Form-based at `/ord?station:|slot:/` path. Username + password fields. Sometimes uses a Java-era styled page.

**Navigation:** Left-side tree panel — hierarchical nav station > network > controller > point. Click to expand nodes. Each node navigates to a "wire sheet" or "px view".

**Levels/Zones:** Usually under a "Building" or site-name node, then floor/level nodes. Zones appear as sub-nodes under each level (e.g. "North Zone", "Zone A", or room names).

**Tips:**
- Tree can be very deep — expand cautiously. `ord` routing in URLs tracks position.
- "Px views" are custom — every site looks different.

---

## Siemens Desigo CC

**Login:** Form at `/manifest.json` base, redirects to `/api/auth`. Modern SPA.

**Navigation:** Left panel with Plant View / Logical View / Management View tabs. Plant View mirrors physical location hierarchy.

**Levels/Zones:** Plant View has Building > Floor > Room hierarchy. Use Plant View (not Logical View) for spatial hierarchy.

**Tips:**
- Right-click context menu on objects reveals "Go to object" for detail page

---

## Schneider Electric EcoStruxure Building Operation (EBO)

**Login:** `/server` path, form-based with domain field sometimes present.

**Navigation:** Left panel with server tree. Expand to find: Network > Controllers > Programs > Points.

**Levels/Zones:** Look for location-based grouping under the site node. May be grouped by controller assignment rather than spatial layout.

**Tips:**
- List views (not graphics) are most structured — look for "List" or "Table" view toggle

---

## Johnson Controls Metasys

**Login:** `/api/v5/login` backend; web UI at `/ui`. SPA with JWT auth.

**Navigation:** Left panel: Network > Site > Equipment > Points hierarchy. "Spaces" view organises by location.

**Levels/Zones:** "Spaces" view has Floor nodes. Zones/rooms appear under each floor. Prefer "Spaces" view over "Equipment" view for spatial hierarchy.

**Tips:**
- JSON API available at `/api/v5/objects` — if accessible, prefer API over UI scraping

---

## Honeywell Enterprise Buildings Integrator (EBI) / SCADA Web

**Login:** Form-based, sometimes IE-era ActiveX legacy UI — may not work in Chrome; flag to user.

**Navigation:** Toolbar-based with system tree on left. Alarm Summary, Point Detail, Graphics views.

**Levels/Zones:** Under site node in system tree. May use "Point Group Display" (PGD) for zone grouping.

**Tips:**
- Older EBI versions may use ActiveX — not extractable; advise user
- If site uses EBI Web Station, navigation is more modern and crawlable

---

## Alerton / Optergy (BACtalk)

**Login:** Form-based at root URL. Username + password fields.

**Navigation:** Main page is a button grid (Mechanical, Floor Layouts, Hydraulics, etc.) with content loaded in an iframe (`page?d=ADVCONTR/<project>/<display_id>`). Navigate directly to displays using `page?dds=ADVCONTR/<project>/<display_id>` to bypass the iframe wrapper.

**Levels/Zones:** "Floor Layouts" page has buttons for each level (Roof, Level 15..1, Ground, B1) linking to floor plan graphics. "Zone Summary" pages may exist with a table of Floor / Zone / Tenancy but these often show tenancy names rather than equipment zones.

**Tips:**
- Content is absolutely positioned — use bounding box coordinates to map text to rows
- Page may never finish loading due to live data polling — wait a few seconds after navigation before reading
- Display IDs follow a pattern: `00005001` = Level 1, `00005002` = Level 2, etc.
- Zone summary tables render level labels as images/positioned elements — crop and read screenshots for best results

---

## Generic / Unknown BMS

If the platform is unrecognised, apply this general approach:

1. **Identify the nav pattern** — sidebar tree, top tabs, or breadcrumb-based
2. **Look for spatial hierarchy** — building, floor, level, wing, zone, area, room
3. **Check the URL scheme** — REST-style `/building/level-1/zone-a` vs hash routing `#/points/123`
4. **Screenshot each level of navigation** as you explore
5. **Ask the user** to clarify which nodes are levels vs zones if unclear
