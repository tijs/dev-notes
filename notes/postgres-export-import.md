# Postgresql: exporting en importing data

### exporting

Exporting a full database, without postgis tables, and fit for use with the pg_restore command.

    pg_dump -Fc --no-acl --no-owner --exclude-table=spatial_ref_sys dbname > pg_dump.dump


### importing

Importing a pg_dump file into a specific database. While cleaning (drop) tables before they are imported.

    pg_restore -d dbname --clean pg_dump.dump