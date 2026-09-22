# Runtime Data

Operational CSV and PNG data is not stored in this repository. In OpenShift it
is mounted from the FMI NFS export and the application reads it through
`HOPS_DATA_DIR` (normally `/mnt/hops/data`).

For local development, provide a compatible data root with these directories:

```text
basins/
metobs/
png/
geojson/
watersheds/
```

Set `HOPS_DATA_DIR` to that root before starting `server.py`. Keep local test
fixtures small and do not commit operational CSV or PNG files.