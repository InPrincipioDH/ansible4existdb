# Recommended Tests 

additional test recommendations for  manual checks

## basic setup: exist-db with restored backup

- [ ] Connect Oxygen editor via webdav to the database: https://www.ediarum.org/docs/set-up/DE/topics/setup/oxygen_mit-db-verbinden.html

## advanced setup: TEI Publisher app

The following tests are recommended after installation of the TEI Publisher App

- [ ] Application can be opened from exist dashboard (only works if no teipublisher.context-path is configured)
- [ ] Application landing page can be reached via reverse proxy (only works if reverse proxy is configured)
- [ ] <app or proxy path>/documentation/customization_table.xml can be opened and displays a table of stylings
- [ ] <app or proxy path>/places displays map at least on most zoomed out layers
- [ ] add a user to the group tei-admin and log in within the app (not on the landing page). A new menu entry with admin options should appear.