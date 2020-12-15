eecs370.org
=============
EECS 370 course web site

The HTML content in `/docs/` is publicly available at TODO: UPDATEME.

<!-- TODO: Update favicon.ico -->

## Contributing
To update the website, modify the files in `/docs/`, commit to `master` and push to GitHub.

Test locally by navigating to http://localhost:8000/
```console
$ cd docs/
$ python3 -m http.server
```

Verify W3C HTML5 compatibility
```console
$ cd docs/
$ html5validator --root .
```

## Calendar
The home page uses the FullCalendar JS library to render the official staff Google Calendar. It is configured to recolor events based on their title, and displays a popup with the event's title and location.

Each semester, remember to update the Google Calendar id (`googleCalendarId`) in the JavaScript code at the end of `index.html`.

If you want to change the colors, modify the `recolorCalendarEvent()` function in `index.html`.

If the Google Calendar API key is not working, follow the guide on this page to obtain a new API key: [https://fullcalendar.io/docs/google-calendar](https://fullcalendar.io/docs/google-calendar).
