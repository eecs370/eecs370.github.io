EECS 370 Course Site Quick Start
=============

## Contributing
To update the website, modify the `index.html` file, commit to `master` and push to GitHub.

```console
$ python3 -m http.server
```
Test locally by navigating to http://localhost:8000/

Verify W3C HTML5 compatibility
```console
$ cd docs/
$ html5validator --root .
```

## Fomantic UI
The site uses https://fomantic-ui.com/ to customize the look.

Theme: default
To modify the CSS variables go to `/semantic/src/themes/default/globals/site.variables`
If you do this you will need to update the build to generate new CSS files:
```console
npx gulp build
```

Generally, looking through the docs and applying the correct keyword in the class attribute will be enough.

## Updating Specs
We are using primer spec to generate specs out of markdown files:
https://github.com/eecs485staff/primer-spec#usage

## Calendar
The home page uses the FullCalendar JS library to render the official staff Google Calendar. It is configured to recolor events based on their title, and displays a popup with the event's title and location.

Each semester, remember to update the Google Calendar id (`googleCalendarId`) in the JavaScript code at the end of `index.html`.

If you want to change the colors, modify the `recolorCalendarEvent()` function in `index.html`.

If the Google Calendar API key is not working, follow the guide on this page to obtain a new API key: [https://fullcalendar.io/docs/google-calendar](https://fullcalendar.io/docs/google-calendar).
