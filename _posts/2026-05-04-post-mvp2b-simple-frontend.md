---
layout: post
title: "Car Management API — MVP 2B: A Simple Vanilla JavaScript Frontend"
description: "Building a plain, readable frontend for the Car Management System — no abstractions, no framework, just straightforward HTML, a handful of CSS rules, and flat JavaScript that anyone can follow."
date: 2026-05-04
category: Architecture
series: Car Management API
series_part: 8
---

*This is part 8 of the Car Management API series. [Part 7](/writing/2026-05-04-post-mvp2b-frontend) covers the API console frontend.*

---

Part 7 of this series covered a developer console for testing the API — a polished single-file tool with syntax highlighting, a status indicator, and a response panel. That frontend was designed to be useful to a developer who already understands what an API is.

This post covers something different: a plain frontend that uses the same API but is aimed at a user who just wants to manage cars. The goal was to keep it simple enough that anyone with a basic understanding of HTML could read the code and follow what is happening — no clever abstractions, nothing that requires prior knowledge of JavaScript patterns.

## The brief

The requirements were deliberately minimal. A form to add a car. A table showing all cars. The ability to edit and delete a row. That is the entire feature set.

There is no routing, no state management, no component model. The page is a single HTML file with a small block of CSS and a handful of JavaScript functions. It opens in a browser and works.

## The CSS

The stylesheet is around thirty lines. It does four things: sets a readable font and a maximum width so the page does not stretch across a wide monitor, adds borders and padding to the table so the data is easy to scan, gives the table rows a light hover colour so it is obvious which row the cursor is on, and hides the edit section by default.

```css
body {
  font-family: Arial, sans-serif;
  max-width: 900px;
  margin: 40px auto;
  padding: 0 20px;
}

table {
  width: 100%;
  border-collapse: collapse;
  margin-top: 16px;
}

th, td {
  border: 1px solid #ccc;
  padding: 8px 10px;
  text-align: left;
}

th { background: #f0f0f0; }
tr:hover { background: #f9f9f9; }
```

`border-collapse: collapse` merges the individual cell borders into single lines rather than leaving a gap between them. Without it the table looks like a grid of separate boxes. `margin: 40px auto` on the body centres the content horizontally and gives it breathing room at the top. These are the two most useful CSS rules to know for basic page layout.

Everything else in the stylesheet is optional polish — the hover colour, the background on the header row — and could be removed without breaking anything.

## Feedback without a UI library

The page needs to tell the user when something has happened: a car was added, a delete failed, the API could not be reached. A full UI library would reach for a toast component or a notification system. Here the same result is achieved with six lines of CSS and a two-line JavaScript function:

```css
.message {
  padding: 8px 12px;
  margin: 10px 0;
  border: 1px solid #bbb;
  background: #f9f9f9;
  display: none;
}
```

```javascript
function showMessage(text) {
  var box = document.getElementById('message');
  box.textContent = text;
  box.style.display = 'block';
  setTimeout(function() { box.style.display = 'none'; }, 3000);
}
```

The element starts hidden. `showMessage` sets its text, makes it visible, and schedules it to hide again after three seconds using `setTimeout`. There is no library involved — this is the browser doing what it already knows how to do.

## Loading and rendering the car list

All cars are loaded by calling `GET /api/Cars` and passing the result to a function that builds the table:

```javascript
function loadCars() {
  fetch(BASE_URL + '/api/Cars')
    .then(function(response) { return response.json(); })
    .then(function(cars) { renderTable(cars); })
    .catch(function() { showMessage('Could not connect to the API.'); });
}
```

`fetch` returns a Promise. The first `.then` converts the HTTP response to a JavaScript object using `.json()`. The second `.then` receives that object — an array of cars — and passes it to `renderTable`. The `.catch` at the end handles network failures.

`renderTable` loops through the array and builds a row for each car:

```javascript
function renderTable(cars) {
  var tbody = document.getElementById('cars-body');

  if (cars.length === 0) {
    tbody.innerHTML = '<tr><td colspan="9">No cars found.</td></tr>';
    return;
  }

  tbody.innerHTML = '';

  for (var i = 0; i < cars.length; i++) {
    var car = cars[i];
    var row = document.createElement('tr');

    row.innerHTML =
      '<td>' + car.make  + '</td>' +
      '<td>' + car.model + '</td>' +
      '<td>' + car.colour + '</td>' +
      '<td>' + car.year  + '</td>' +
      '<td>£' + car.price.toLocaleString() + '</td>' +
      '<td>' + car.mileage.toLocaleString() + '</td>' +
      '<td>' + FUEL_TYPES[car.fuelType]  + '</td>' +
      '<td>' + GEARBOXES[car.gearbox]    + '</td>' +
      '<td><button onclick="editCar(\'' + car.id + '\')">Edit</button>' +
           '<button onclick="deleteCar(\'' + car.id + '\')">Delete</button></td>';

    tbody.appendChild(row);
  }
}
```

Setting `tbody.innerHTML = ''` before the loop clears any existing rows, which means calling `loadCars` again after an add or delete always shows a fresh result.

Two things are worth noting in the row construction. `toLocaleString()` on a number adds comma separators automatically — `30000` becomes `30,000` and `15000` becomes `15,000`. And the enum values from the API — integers — are converted back to readable labels using lookup arrays:

```javascript
const BODY_TYPES = ['Saloon', 'Hatchback', 'Estate', 'Coupe', 'Convertible', 'SUV', 'MPV', 'Van'];
const GEARBOXES  = ['Manual', 'Automatic'];
const FUEL_TYPES = ['Petrol', 'Diesel', 'Electric', 'Hybrid', 'Hydrogen'];
```

The position in the array matches the integer value the API sends. `FUEL_TYPES[0]` is `'Petrol'`, `FUEL_TYPES[2]` is `'Electric'`. The table never shows a raw number to the user.

## Adding a car

The add form collects values from named inputs and builds a plain JavaScript object:

```javascript
function addCar() {
  var car = {
    make:       document.getElementById('add-make').value,
    model:      document.getElementById('add-model').value,
    year:       parseInt(document.getElementById('add-year').value) || 0,
    price:      parseFloat(document.getElementById('add-price').value) || 0,
    gearbox:    parseInt(document.getElementById('add-gearbox').value),
    // ...remaining fields
  };

  fetch(BASE_URL + '/api/Cars', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(car)
  })
    .then(function(response) { return response.json(); })
    .then(function(result) {
      if (result) {
        showMessage('Car added successfully.');
        loadCars();
      } else {
        showMessage('Failed to add car.');
      }
    });
}
```

`parseInt(value) || 0` handles the case where a number field is left blank. An empty string passed to `parseInt` returns `NaN`, and `NaN || 0` evaluates to `0`. This prevents the API receiving a malformed value without requiring any extra validation logic.

`JSON.stringify(car)` converts the object to a JSON string for the request body. `Content-Type: application/json` tells the API what format to expect. After a successful add, `loadCars()` is called again so the table updates immediately.

## Editing a car

Clicking Edit on a row calls `editCar` with that car's ID. The function fetches the car by ID, populates the edit form with its current values, and scrolls the form into view:

```javascript
function editCar(id) {
  fetch(BASE_URL + '/api/Cars/' + id)
    .then(function(response) { return response.json(); })
    .then(function(car) {
      document.getElementById('edit-id').value    = car.id;
      document.getElementById('edit-make').value  = car.make;
      document.getElementById('edit-model').value = car.model;
      // ...remaining fields

      document.getElementById('edit-section').style.display = 'block';
      document.getElementById('edit-section').scrollIntoView();
    });
}
```

The ID is stored in a hidden input field so it is available when the save button is clicked without needing to be displayed to the user.

`scrollIntoView()` is a built-in browser method that scrolls the page until the element is visible. The edit form sits below the table, so without this call a user who had scrolled down to find a car would need to scroll back up to realise the form had appeared.

Saving calls `PUT /api/Cars/{id}` with the updated object, and cancelling simply hides the edit section again.

## Deleting a car

Delete uses the browser's built-in `confirm` dialog before firing the request:

```javascript
function deleteCar(id) {
  if (!confirm('Delete this car?')) return;

  fetch(BASE_URL + '/api/Cars/' + id, { method: 'DELETE' })
    .then(function(response) { return response.json(); })
    .then(function(result) {
      if (result) {
        showMessage('Car deleted.');
        loadCars();
      }
    });
}
```

`confirm` blocks execution and returns `true` if the user clicks OK and `false` if they click Cancel. Returning early on `false` means the fetch never fires. It is not a sophisticated pattern, but for a simple frontend it avoids an accidental deletion without introducing a custom modal component.

## Why `var` and `.then()` instead of `const` and `async/await`

The code uses `var` throughout and writes asynchronous calls using `.then()` chains rather than `async/await`. Both of these are deliberate choices.

`const` and `let` are the modern preference, but `var` is what someone encounters first when learning JavaScript. It behaves predictably in the flat function scope used here and introduces no confusion about block scoping. A beginner reading this code does not need to understand the difference between `var`, `let`, and `const` to follow what it is doing.

`async/await` makes asynchronous code read like synchronous code, which is useful when working with complex flows. Here every operation is a single fetch call followed by a result check — the `.then()` chain makes the sequence explicit and visible. The data flows left to right through each `.then`, which is easier to reason about than jumping into and out of async functions when the operations are this straightforward.

Both decisions prioritise readability over idiom.

## What it does not do

There is no client-side validation. Empty fields are sent to the API. There is no loading indicator while a request is in flight. There is no error detail beyond a generic message. The list does not load automatically on page open — the user has to click Refresh List.

These are all reasonable additions, and each one is small enough to be a learning exercise in itself. The code as it stands is a starting point — something that works and can be understood completely before anything is added to it.

---

*Next: integration tests using TestContainers.*
