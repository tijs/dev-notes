# Postgresql: exporteren en importeren

### exporteren

Exporteer de volledige database, zonder eventuele postgis tabellen, geschikt voor pg_restore.

    pg_dump -Fc --no-acl --no-owner --exclude-table=spatial_ref_sys dbname > pg_dump.dump


### importeren

Importeer een pg_dump bestand in een specifieke database. Clean (drop) tabellen voor de import.

    pg_restore -d dbname --clean pg_dump.dump