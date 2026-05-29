# Collapsible Genealogy Tree Editor — Improved Single-File Version

Save this as `genealogy.html` and open it in a browser.

This version implements a more durable architecture than the first draft:

- One central `people` table instead of deeply duplicated nested objects
- People linked by IDs
- Add child
- Add parent
- Add spouse
- Editable person fields
- Notes
- Search
- Collapsible person cards
- Visual relationship lines using SVG
- JSON export
- JSON import
- Browser local save/load
- Human-readable data model
- Copious comments throughout

This is still intentionally plain HTML/CSS/JavaScript with no framework and no external libraries.

---

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Genealogy Tree Editor</title>

<style>

/* =========================================================
   PAGE-WIDE STYLING
   ========================================================= */

body {
    font-family: Arial, sans-serif;
    margin: 20px;
    background: #f3f3f3;
    color: #222;
}

h1 {
    margin-bottom: 5px;
}

button {
    cursor: pointer;
    padding: 5px 8px;
    margin: 2px;
}

input,
textarea,
select {
    font-family: inherit;
}

/* =========================================================
   TOOLBAR
   ========================================================= */

.toolbar {
    background: white;
    border: 1px solid #bbb;
    border-radius: 8px;
    padding: 10px;
    margin-bottom: 15px;
}

.toolbar-row {
    margin-bottom: 8px;
}

.toolbar input[type="text"] {
    padding: 6px;
    width: 300px;
}

/* =========================================================
   TREE AREA
   ========================================================= */

#treeWrapper {
    position: relative;
    background: #fafafa;
    border: 1px solid #bbb;
    border-radius: 8px;
    padding: 20px;
    overflow: auto;
    min-height: 400px;
}

/* =========================================================
   SVG RELATIONSHIP LAYER

   This SVG sits behind the person cards and draws lines between
   parent and child cards.
   ========================================================= */

#relationshipLines {
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
    z-index: 1;
}

/* =========================================================
   TREE CONTENT LAYER

   Person cards sit above the SVG lines.
   ========================================================= */

#treeContainer {
    position: relative;
    z-index: 2;
}

/* =========================================================
   PERSON CARD
   ========================================================= */

.person-card {
    background: white;
    border: 1px solid #aaa;
    border-radius: 8px;
    margin: 12px 0 12px 35px;
    padding: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.12);
    max-width: 760px;
}

/* A highlighted card is used for search results. */
.person-card.highlighted {
    outline: 3px solid #f0b400;
}

.person-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: #e8e8e8;
    padding: 6px;
    border-radius: 6px;
}

.person-name {
    font-weight: bold;
    font-size: 18px;
}

.person-subtitle {
    font-size: 12px;
    color: #555;
}

.person-body {
    margin-top: 10px;
}

.field-row {
    margin-bottom: 8px;
}

.field-row label {
    display: inline-block;
    width: 135px;
    font-weight: bold;
}

.field-row input {
    width: 350px;
    padding: 4px;
}

.field-row textarea {
    width: 350px;
    height: 80px;
    padding: 4px;
    vertical-align: top;
}

.children-container {
    margin-left: 35px;
    border-left: 2px solid #ccc;
    padding-left: 10px;
}

.spouse-list,
.parent-list,
.child-list {
    font-size: 13px;
    color: #444;
    margin: 5px 0;
}

.small-note {
    font-size: 12px;
    color: #666;
}

.hidden {
    display: none;
}

</style>
</head>
<body>

<h1>Genealogy Tree Editor</h1>
<p class="small-note">
    Single-file local genealogy editor using JSON-style relational data.
</p>

<!-- =====================================================
     TOOLBAR
     ===================================================== -->

<div class="toolbar">

    <div class="toolbar-row">
        <button onclick="addRootPerson()">Add New Root Person</button>
        <button onclick="saveToBrowser()">Save to Browser</button>
        <button onclick="loadFromBrowser()">Load from Browser</button>
        <button onclick="clearBrowserSave()">Clear Browser Save</button>
    </div>

    <div class="toolbar-row">
        <button onclick="exportJSON()">Export JSON</button>
        <button onclick="document.getElementById('importFile').click()">Import JSON</button>
        <input id="importFile" type="file" accept="application/json,.json" class="hidden" onchange="importJSON(event)">
    </div>

    <div class="toolbar-row">
        <input id="searchBox" type="text" placeholder="Search name, location, notes..." oninput="renderTree()">
        <button onclick="document.getElementById('searchBox').value=''; renderTree();">Clear Search</button>
    </div>

</div>

<!-- =====================================================
     TREE WRAPPER

     This contains both:
     1. SVG relationship lines
     2. Actual person cards
     ===================================================== -->

<div id="treeWrapper">
    <svg id="relationshipLines"></svg>
    <div id="treeContainer"></div>
</div>

<script>

/* =========================================================
   IMPORTANT ARCHITECTURE NOTE
   =========================================================

   The first/simple version used deeply nested objects:

       person.children = [ whole child object, whole child object ]

   That becomes a problem because a person can be reached through
   multiple family paths. Duplicating full objects can cause stale data.

   This version uses a more database-like layout:

       genealogy.people[personId] = person object

   And relationships are arrays of IDs:

       person.children = ["p123", "p456"]
       person.parents  = ["p789", "p999"]
       person.spouses  = ["p555"]

   This is much closer to how genealogy software and databases work.

   It also makes import/export cleaner.

   ========================================================= */


/* =========================================================
   MAIN DATA STRUCTURE
   =========================================================

   genealogy.roots:
       List of people to display as top-level starting points.

   genealogy.people:
       A lookup table of all people by ID.

   Example:

       genealogy.people["p1001"].firstName

   ========================================================= */

let genealogy = {
    roots: [],
    people: {}
};


/* =========================================================
   COLLAPSE STATE
   =========================================================

   This is UI-only data.

   It is kept separate from genealogy data so exporting genealogy JSON
   does not necessarily have to include interface state.

   collapsedPeople[personId] = true means that person's body is hidden.

   ========================================================= */

let collapsedPeople = {};


/* =========================================================
   ID GENERATOR
   =========================================================

   IDs are strings instead of numbers because strings are easier to
   recognize and safer as object keys.

   This creates IDs such as:

       p837462910

   For serious long-term use, a UUID generator would be better, but this
   is perfectly adequate for a local personal genealogy tool.

   ========================================================= */

function generatePersonId() {
    return "p" + Math.floor(Math.random() * 1000000000);
}


/* =========================================================
   CREATE A NEW PERSON OBJECT
   =========================================================

   This is the one place where default person fields are defined.

   If you want to add a new field globally, this is one place to modify.

   Example additions:

       occupation: "",
       baptismDate: "",
       burialLocation: "",
       sourceCitation: ""

   You would then add matching editor fields in renderPersonCard().

   ========================================================= */

function createPerson(firstName = "New", lastName = "Person") {
    const id = generatePersonId();

    return {
        id: id,
        firstName: firstName,
        lastName: lastName,
        birthDate: "",
        birthLocation: "",
        deathDate: "",
        deathLocation: "",
        notes: "",
        parents: [],
        children: [],
        spouses: []
    };
}


/* =========================================================
   INITIAL SAMPLE DATA
   =========================================================

   This gives the page something useful to show when first opened.

   You can delete this and start blank if you prefer.

   ========================================================= */

function createSampleTree() {

    const root = createPerson("Root", "Person");
    genealogy.people[root.id] = root;
    genealogy.roots.push(root.id);

    renderTree();
}


/* =========================================================
   ADD ROOT PERSON
   =========================================================

   A root person is simply a top-level visible starting point.

   This allows multiple disconnected family groups in one file.

   ========================================================= */

function addRootPerson() {
    const person = createPerson("New", "Root");
    genealogy.people[person.id] = person;
    genealogy.roots.push(person.id);
    renderTree();
}


/* =========================================================
   ADD CHILD
   =========================================================

   This creates a new child person, links parent -> child,
   and also links child -> parent.

   This is bidirectional relationship maintenance.

   ========================================================= */

function addChild(parentId) {
    const parent = genealogy.people[parentId];
    if (!parent) return;

    const child = createPerson("New", "Child");
    genealogy.people[child.id] = child;

    linkParentChild(parentId, child.id);

    renderTree();
}


/* =========================================================
   ADD PARENT
   =========================================================

   This creates a new parent person and links parent -> child.

   If the child is currently a root, the new parent is added as a root
   so the ancestry can be displayed above/around that child.

   ========================================================= */

function addParent(childId) {
    const child = genealogy.people[childId];
    if (!child) return;

    const parent = createPerson("New", "Parent");
    genealogy.people[parent.id] = parent;

    linkParentChild(parent.id, childId);

    // Add new parent as a root so the user can see the upward extension.
    if (!genealogy.roots.includes(parent.id)) {
        genealogy.roots.push(parent.id);
    }

    renderTree();
}


/* =========================================================
   ADD SPOUSE
   =========================================================

   Spouse links are also bidirectional.

   spouse A contains spouse B's ID.
   spouse B contains spouse A's ID.

   ========================================================= */

function addSpouse(personId) {
    const person = genealogy.people[personId];
    if (!person) return;

    const spouse = createPerson("New", "Spouse");
    genealogy.people[spouse.id] = spouse;

    linkSpouses(personId, spouse.id);

    renderTree();
}


/* =========================================================
   LINK PARENT AND CHILD
   =========================================================

   This function prevents duplicated relationship links.

   Instead of manually pushing IDs all over the code, use this helper.

   ========================================================= */

function linkParentChild(parentId, childId) {
    const parent = genealogy.people[parentId];
    const child = genealogy.people[childId];

    if (!parent || !child) return;

    if (!parent.children.includes(childId)) {
        parent.children.push(childId);
    }

    if (!child.parents.includes(parentId)) {
        child.parents.push(parentId);
    }
}


/* =========================================================
   LINK SPOUSES
   ========================================================= */

function linkSpouses(personAId, personBId) {
    const a = genealogy.people[personAId];
    const b = genealogy.people[personBId];

    if (!a || !b) return;

    if (!a.spouses.includes(personBId)) {
        a.spouses.push(personBId);
    }

    if (!b.spouses.includes(personAId)) {
        b.spouses.push(personAId);
    }
}


/* =========================================================
   DELETE PERSON
   =========================================================

   Deletion is now implemented carefully.

   Steps:
   1. Remove this person's ID from every other person's relationship lists
   2. Remove from root list
   3. Delete the person object itself

   This does NOT delete descendants.

   That is deliberate.

   In genealogy data, deleting a person should not automatically delete
   children, parents, or spouses unless explicitly requested.

   ========================================================= */

function deletePerson(personId) {
    const person = genealogy.people[personId];
    if (!person) return;

    const fullName = getFullName(person);

    if (!confirm(`Delete ${fullName}?\n\nThis removes the person but does not delete relatives.`)) {
        return;
    }

    Object.values(genealogy.people).forEach(function(otherPerson) {
        otherPerson.parents = otherPerson.parents.filter(id => id !== personId);
        otherPerson.children = otherPerson.children.filter(id => id !== personId);
        otherPerson.spouses = otherPerson.spouses.filter(id => id !== personId);
    });

    genealogy.roots = genealogy.roots.filter(id => id !== personId);

    delete genealogy.people[personId];

    renderTree();
}


/* =========================================================
   GET FULL NAME
   ========================================================= */

function getFullName(person) {
    return `${person.firstName || ""} ${person.lastName || ""}`.trim() || "Unnamed Person";
}


/* =========================================================
   RENDER WHOLE TREE
   ========================================================= */

function renderTree() {
    const container = document.getElementById("treeContainer");
    container.innerHTML = "";

    const searchText = document.getElementById("searchBox").value.toLowerCase().trim();

    if (genealogy.roots.length === 0) {
        const emptyMessage = document.createElement("p");
        emptyMessage.textContent = "No people yet. Click 'Add New Root Person' to begin.";
        container.appendChild(emptyMessage);
        drawRelationshipLines();
        return;
    }

    const rendered = new Set();

    genealogy.roots.forEach(function(rootId) {
        if (genealogy.people[rootId]) {
            renderPersonCard(rootId, container, rendered, searchText);
        }
    });

    // Relationship lines need DOM positions, so draw after rendering.
    setTimeout(drawRelationshipLines, 0);
}


/* =========================================================
   RENDER PERSON CARD
   =========================================================

   This function recursively renders the selected person and descendants.

   rendered:
       A Set used to prevent infinite loops if relationships become cyclic.

   searchText:
       Current search input.

   ========================================================= */

function renderPersonCard(personId, parentElement, rendered, searchText) {
    const person = genealogy.people[personId];
    if (!person) return;

    /* -----------------------------------------------------
       Prevent infinite loops.

       This matters because genealogy is not always a simple tree.
       Cousin marriages, duplicate roots, and data mistakes can create
       circular graph paths.
       ----------------------------------------------------- */

    if (rendered.has(personId)) {
        const duplicateNotice = document.createElement("div");
        duplicateNotice.className = "person-card";
        duplicateNotice.textContent = `${getFullName(person)} is already shown elsewhere.`;
        parentElement.appendChild(duplicateNotice);
        return;
    }

    rendered.add(personId);

    const card = document.createElement("div");
    card.className = "person-card";
    card.dataset.personId = personId;

    if (personMatchesSearch(person, searchText)) {
        card.classList.add("highlighted");
    }

    /* =====================================================
       HEADER
       ===================================================== */

    const header = document.createElement("div");
    header.className = "person-header";

    const titleBlock = document.createElement("div");

    const nameDiv = document.createElement("div");
    nameDiv.className = "person-name";
    nameDiv.textContent = getFullName(person);

    const subtitle = document.createElement("div");
    subtitle.className = "person-subtitle";
    subtitle.textContent = `${person.birthDate || "?"} – ${person.deathDate || "?"}`;

    titleBlock.appendChild(nameDiv);
    titleBlock.appendChild(subtitle);

    const buttonBlock = document.createElement("div");

    const collapseButton = document.createElement("button");
    collapseButton.textContent = collapsedPeople[personId] ? "Expand" : "Collapse";
    collapseButton.onclick = function() {
        collapsedPeople[personId] = !collapsedPeople[personId];
        renderTree();
    };

    const addParentButton = document.createElement("button");
    addParentButton.textContent = "Add Parent";
    addParentButton.onclick = function() {
        addParent(personId);
    };

    const addChildButton = document.createElement("button");
    addChildButton.textContent = "Add Child";
    addChildButton.onclick = function() {
        addChild(personId);
    };

    const addSpouseButton = document.createElement("button");
    addSpouseButton.textContent = "Add Spouse";
    addSpouseButton.onclick = function() {
        addSpouse(personId);
    };

    const deleteButton = document.createElement("button");
    deleteButton.textContent = "Delete";
    deleteButton.onclick = function() {
        deletePerson(personId);
    };

    buttonBlock.appendChild(collapseButton);
    buttonBlock.appendChild(addParentButton);
    buttonBlock.appendChild(addChildButton);
    buttonBlock.appendChild(addSpouseButton);
    buttonBlock.appendChild(deleteButton);

    header.appendChild(titleBlock);
    header.appendChild(buttonBlock);

    card.appendChild(header);

    /* =====================================================
       BODY
       ===================================================== */

    const body = document.createElement("div");
    body.className = "person-body";

    if (collapsedPeople[personId]) {
        body.style.display = "none";
    }

    body.appendChild(createInputField("First Name", person.firstName, function(value) {
        person.firstName = value;
        renderTree();
    }));

    body.appendChild(createInputField("Last Name", person.lastName, function(value) {
        person.lastName = value;
        renderTree();
    }));

    body.appendChild(createInputField("Birth Date", person.birthDate, function(value) {
        person.birthDate = value;
        renderTree();
    }));

    body.appendChild(createInputField("Birth Location", person.birthLocation, function(value) {
        person.birthLocation = value;
    }));

    body.appendChild(createInputField("Death Date", person.deathDate, function(value) {
        person.deathDate = value;
        renderTree();
    }));

    body.appendChild(createInputField("Death Location", person.deathLocation, function(value) {
        person.deathLocation = value;
    }));

    body.appendChild(createTextAreaField("Notes", person.notes, function(value) {
        person.notes = value;
    }));

    /* =====================================================
       RELATIONSHIP SUMMARY LISTS
       ===================================================== */

    body.appendChild(createRelationshipSummary("Parents", person.parents));
    body.appendChild(createRelationshipSummary("Spouses", person.spouses));
    body.appendChild(createRelationshipSummary("Children", person.children));

    /* =====================================================
       CHILDREN DISPLAY
       ===================================================== */

    const childrenContainer = document.createElement("div");
    childrenContainer.className = "children-container";

    person.children.forEach(function(childId) {
        renderPersonCard(childId, childrenContainer, rendered, searchText);
    });

    body.appendChild(childrenContainer);
    card.appendChild(body);
    parentElement.appendChild(card);
}


/* =========================================================
   CREATE TEXT INPUT FIELD
   ========================================================= */

function createInputField(labelText, value, onInput) {
    const row = document.createElement("div");
    row.className = "field-row";

    const label = document.createElement("label");
    label.textContent = labelText;

    const input = document.createElement("input");
    input.type = "text";
    input.value = value || "";

    input.oninput = function() {
        onInput(input.value);
    };

    row.appendChild(label);
    row.appendChild(input);

    return row;
}


/* =========================================================
   CREATE TEXTAREA FIELD
   ========================================================= */

function createTextAreaField(labelText, value, onInput) {
    const row = document.createElement("div");
    row.className = "field-row";

    const label = document.createElement("label");
    label.textContent = labelText;

    const textarea = document.createElement("textarea");
    textarea.value = value || "";

    textarea.oninput = function() {
        onInput(textarea.value);
    };

    row.appendChild(label);
    row.appendChild(textarea);

    return row;
}


/* =========================================================
   CREATE RELATIONSHIP SUMMARY
   ========================================================= */

function createRelationshipSummary(labelText, ids) {
    const div = document.createElement("div");
    div.className = "small-note";

    const names = ids
        .map(id => genealogy.people[id])
        .filter(Boolean)
        .map(person => getFullName(person));

    div.textContent = `${labelText}: ${names.length ? names.join(", ") : "None"}`;

    return div;
}


/* =========================================================
   SEARCH MATCHING
   ========================================================= */

function personMatchesSearch(person, searchText) {
    if (!searchText) return false;

    const blob = [
        person.firstName,
        person.lastName,
        person.birthDate,
        person.birthLocation,
        person.deathDate,
        person.deathLocation,
        person.notes
    ].join(" ").toLowerCase();

    return blob.includes(searchText);
}


/* =========================================================
   DRAW RELATIONSHIP LINES
   =========================================================

   This is a lightweight graphical display.

   It draws lines from each visible parent card to each visible child card.

   This is not a full chart layout engine. It is a simple visual connector
   layer laid over the recursive card layout.

   A more advanced future version could use:

   - horizontal pedigree chart
   - force-directed graph
   - fan chart
   - canvas renderer
   - SVG nodes instead of HTML cards

   ========================================================= */

function drawRelationshipLines() {
    const wrapper = document.getElementById("treeWrapper");
    const svg = document.getElementById("relationshipLines");

    const wrapperRect = wrapper.getBoundingClientRect();

    svg.setAttribute("width", wrapper.scrollWidth);
    svg.setAttribute("height", wrapper.scrollHeight);
    svg.innerHTML = "";

    Object.values(genealogy.people).forEach(function(person) {
        person.children.forEach(function(childId) {
            const parentCard = document.querySelector(`[data-person-id="${person.id}"]`);
            const childCard = document.querySelector(`[data-person-id="${childId}"]`);

            if (!parentCard || !childCard) return;

            const parentRect = parentCard.getBoundingClientRect();
            const childRect = childCard.getBoundingClientRect();

            const x1 = parentRect.left - wrapperRect.left + parentRect.width / 2 + wrapper.scrollLeft;
            const y1 = parentRect.bottom - wrapperRect.top + wrapper.scrollTop;

            const x2 = childRect.left - wrapperRect.left + childRect.width / 2 + wrapper.scrollLeft;
            const y2 = childRect.top - wrapperRect.top + wrapper.scrollTop;

            const line = document.createElementNS("http://www.w3.org/2000/svg", "line");
            line.setAttribute("x1", x1);
            line.setAttribute("y1", y1);
            line.setAttribute("x2", x2);
            line.setAttribute("y2", y2);
            line.setAttribute("stroke", "#999");
            line.setAttribute("stroke-width", "2");

            svg.appendChild(line);
        });
    });
}


/* =========================================================
   SAVE TO BROWSER LOCAL STORAGE
   ========================================================= */

function saveToBrowser() {
    localStorage.setItem("genealogyData", JSON.stringify(genealogy, null, 2));
    alert("Saved to this browser.");
}


/* =========================================================
   LOAD FROM BROWSER LOCAL STORAGE
   ========================================================= */

function loadFromBrowser() {
    const saved = localStorage.getItem("genealogyData");

    if (!saved) {
        alert("No browser save found.");
        return;
    }

    genealogy = JSON.parse(saved);
    renderTree();
    alert("Loaded from browser.");
}


/* =========================================================
   CLEAR BROWSER SAVE
   ========================================================= */

function clearBrowserSave() {
    if (!confirm("Clear saved browser copy?")) return;

    localStorage.removeItem("genealogyData");
    alert("Browser save cleared.");
}


/* =========================================================
   EXPORT JSON FILE
   =========================================================

   This downloads your genealogy data as a .json file.

   This is the recommended working data format for this tool.

   ========================================================= */

function exportJSON() {
    const json = JSON.stringify(genealogy, null, 2);
    const blob = new Blob([json], { type: "application/json" });
    const url = URL.createObjectURL(blob);

    const a = document.createElement("a");
    a.href = url;
    a.download = "genealogy-data.json";
    a.click();

    URL.revokeObjectURL(url);
}


/* =========================================================
   IMPORT JSON FILE
   ========================================================= */

function importJSON(event) {
    const file = event.target.files[0];
    if (!file) return;

    const reader = new FileReader();

    reader.onload = function(e) {
        try {
            const imported = JSON.parse(e.target.result);

            if (!imported.people || !imported.roots) {
                alert("Invalid genealogy JSON file.");
                return;
            }

            genealogy = imported;
            renderTree();
            alert("JSON imported.");
        }
        catch (err) {
            alert("Could not parse JSON file.");
        }
    };

    reader.readAsText(file);
}


/* =========================================================
   REDRAW LINES WHEN WINDOW SIZE CHANGES
   ========================================================= */

window.addEventListener("resize", drawRelationshipLines);


/* =========================================================
   STARTUP
   ========================================================= */

createSampleTree();

</script>
</body>
</html>
```

---

# What This Version Implements

## 1. Relational Linking

The data now uses ID-based relationships:

```javascript
person.children = ["p123", "p456"];
person.parents = ["p789"];
person.spouses = ["p999"];
```

This is much better than storing entire child objects inside parent objects.

It prevents duplicated people and makes future features much cleaner.

---

## 2. Graphical Display

The visual connectors are drawn with an SVG layer behind the cards.

This gives you parent-child relationship lines without needing an external charting library.

It is still not a full ancestry chart layout engine, but it is a major step toward one.

---

## 3. JSON Import/Export

This makes the genealogy data portable.

You can export:

```text
genealogy-data.json
```

Then later import it back into the page.

---

## 4. Multiple Root People

You can now have disconnected family branches.

That matters because real genealogy research often starts with partial fragments.

---

## 5. Spouses

Spouse linking is included.

Children are still attached directly to parent records, not to couple/family units. That is acceptable for this stage, but a more GEDCOM-like model would eventually add family/marriage records.

---

# Most Important Future Upgrade

The next serious architectural upgrade would be to add a separate `families` table.

Right now:

```javascript
people[p001].children = [p003]
people[p002].children = [p003]
```

A more formal genealogy structure would be:

```javascript
families[f001] = {
    spouses: ["p001", "p002"],
    children: ["p003", "p004"],
    marriageDate: "",
    marriageLocation: "",
    notes: ""
}
```

That is closer to GEDCOM and better for:

- Multiple marriages
- Half-siblings
- Stepchildren
- Adoption
- Unknown parent
- Marriage/divorce dates
- Source citations

That should probably be the next version if you want this to become a genuinely usable genealogy tool rather than just a family tree sketchpad.

