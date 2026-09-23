# Deploy HOPS Webservice on OpenShift

Working runbook for publishing this repository to the live HOPS OpenShift deployment.

## Production Details

| Item | Value |
| --- | --- |
| OpenShift API | Obtain from the FMI OpenShift console |
| Project | `hops-webservice` |
| Public route | `https://hops-webservice-hops-webservice.apps.ock.fmi.fi` |
| Git repository | `https://github.com/ikonenjaakko-lab/hops_webservice.git` |
| Build branch | `master` |
| BuildConfig/ImageStream | `hops-webservice` |
| Deployment/Service/Route | `hops-webservice` |
| Container port | `8000` |
| Health endpoint | `/health` |
| NFS mount | FMI-managed read-only mount -> `/mnt/hops` |
| Application data | `/mnt/hops/data` |

The active OpenShift resource name is `hops-webservice`. The former `hops-v2` name is obsolete.

## Rules and Prerequisites

- Have `oc`, Git, Python, and `curl.exe` available.
- Authenticate to OpenShift and select the `hops-webservice` project before acting.
- Never put tokens, passwords, private keys, or operational data in Git, this file, or chat.
- A Git push does not deploy anything by itself. A build and rollout are required.
- The BuildConfig builds `master`, so push the intended commit there first.
- Use an immutable image reference for a release when possible; `latest` is not a release identifier.

```powershell
# Use the login command provided by the FMI OpenShift web console.
oc login <fmi-openshift-api>
oc whoami
oc project hops-webservice
oc project -q
oc auth can-i create builds.build.openshift.io
oc auth can-i update deployments.apps
```

The project command must print `hops-webservice`.

## Versioned OpenShift Manifests

- `deploy/buildconfig.local.yaml` defines the local-only `hops-webservice` ImageStream and BuildConfig.
- `deploy/openshift.local.yaml` defines the local-only Deployment, Service, Route, and read-only NFS volume.

Apply these when creating or intentionally reconciling OpenShift resources. They are not needed for every frontend release:

```powershell
oc apply -f deploy/buildconfig.local.yaml
oc apply -f deploy/openshift.local.yaml
oc rollout status deployment/hops-webservice --timeout=180s
```

Review the live resources before applying the application manifest:

```powershell
oc get deployment hops-webservice -o yaml
oc get buildconfig hops-webservice -o yaml
```

## Normal Frontend or Application Release

There is no npm, Vite, or JavaScript build step. Changes to `frontend/`, `config/`, `content/`, `server.py`, or `Containerfile` require a new image.

### 1. Edit and test locally

```powershell
git switch master
git pull --ff-only
python .\server.py
```

Open `http://127.0.0.1:8000` and test the change. For a title change, edit `config/app.json`: `subtitle` is the large heading and `title` is the smaller text below it.

Review the change:

```powershell
git status
git diff --check
git diff
```

### 2. Commit and push

```powershell
git add <changed-files>
git commit -m "Describe the visible change"
git push origin master
git status -sb
```

### 3. Build the image

```powershell
oc project hops-webservice
oc start-build hops-webservice --follow
oc get builds
oc get imagestreamtag hops-webservice:latest
```

The BuildConfig checks out GitHub `master`, uses the root `Containerfile`, and publishes to the internal ImageStream.

### 4. Deploy the exact built image

```powershell
$imageRef = oc get imagestreamtag hops-webservice:latest `
  -o jsonpath='{.image.dockerImageReference}'
Write-Host $imageRef
oc set image deployment/hops-webservice "hops-webservice=$imageRef"
oc rollout status deployment/hops-webservice --timeout=180s
```

Record the Git commit, build name, image reference/digest, and rollout result.

### 5. Validate live

```powershell
oc get deployment,pods,service,route
oc logs deployment/hops-webservice --tail=100
$routeHost = oc get route hops-webservice -o jsonpath='{.spec.host}'
$routeUrl = "https://$routeHost"
curl.exe -i "$routeUrl/health"
curl.exe -i "$routeUrl/api/config"
curl.exe -s -o NUL -w "%{http_code}" "$routeUrl/"
curl.exe -s -o NUL -w "%{http_code}" "$routeUrl/api/timeseries/6501700"
```

Expected: the pod is Ready with no unexpected restarts, all four endpoints return HTTP 200, and the changed behavior is visible after a hard refresh or in a private browser window. Check one map overlay, one basin graph, one time-series request, and the browser console/network panel.

## Data-Only Changes

Files under the NFS data directory are read directly and do not require an image build or rollout:

```text
/mnt/hops/data/
  basins/
  geojson/
  metobs/
  png/
  watersheds/
```

Verify the mount after an update:

```powershell
oc exec deployment/hops-webservice -- ls -la /mnt/hops/data
oc exec deployment/hops-webservice -- find /mnt/hops/data -maxdepth 2 -type d
```

Preserve the filename and directory contracts in `server.py` and `config/app.json`. Write replacement files to a temporary name and rename them into place where possible.

For streamflow model CSVs, a missing file makes that model unavailable. A model whose `value` entries are all `-99998` or lower is also treated as unavailable; its streamflow checkbox is disabled and it is excluded from the graph and statistics. At least one value greater than `-99998` is required for the model to be selectable.

### Weather station configuration

Station count, labels, coordinates, types, and CSV filenames are read from the external `observation-stations.json` file. In OpenShift this file is expected at `/mnt/hops/config/observation-stations.json`. Replace that file on the mounted data volume to add, remove, or move stations without rebuilding the image or rolling out the Deployment. The application falls back to the bundled template if the external file is absent.

The file must contain a JSON array with entries shaped like:

```json
[
  {
    "id": "example_station",
    "label": "Example Station",
    "lat": 67.0,
    "lon": 26.0,
    "type": "temperature",
    "dataFile": "fmi_tempc_example.csv"
  }
]
```

After changing it, refresh the application and verify `/api/config`, the station markers, and the corresponding files under `data/metobs/`.

## Basemap

The current watermark-free settings are:

```text
HOPS_TILE_URL=https://tile.openstreetmap.org/{z}/{x}/{y}.png
HOPS_ENABLE_HILLSHADE=false
```

The former CARTO dark URL displayed `API KEY REQUIRED`. Do not restore it without an approved provider, attribution, credentials, hostname restrictions, quotas, and availability. OpenStreetMap tiles require visible attribution and normal interactive use.

## Rollback

Inspect known images:

```powershell
oc rollout history deployment/hops-webservice
oc get replicasets -l app=hops-webservice `
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,CREATED:.metadata.creationTimestamp
```

Restore a previously verified immutable image reference:

```powershell
oc set image deployment/hops-webservice `
  "hops-webservice=<previous-working-image-reference>"
oc rollout status deployment/hops-webservice --timeout=180s
```

Validate `/health` and the live application after rollback. Do not rely only on `oc rollout undo` when history refers to mutable `latest`.

## Troubleshooting

```powershell
# Change not visible
git log -1 --oneline
oc get builds
oc get imagestreamtag hops-webservice:latest
oc get deployment hops-webservice -o jsonpath='{.spec.template.spec.containers[0].image}'

# Pod or route failure
oc get pods -o wide
oc describe pod -l app=hops-webservice
oc logs deployment/hops-webservice
oc get events --sort-by=.lastTimestamp
oc get service hops-webservice
oc get endpointslice -l kubernetes.io/service-name=hops-webservice
```

For map failures, inspect `runtime-config.js`, browser network requests, pod logs, NFS GeoJSON paths, and external tile requests separately.

## Still Missing or To Be Confirmed

- Current OpenShift login identity/service account and release permissions.
- Whether the public route is intentionally anonymous or needs authentication.
- A GitHub webhook/OpenShift trigger; currently builds are started manually.
- A formal release record and retention of the previous rollback image.
- Monitoring and alert ownership for probes, route, NFS, and build failures.
- Whether the diagnostic `test-nfs-mount` Deployment can be removed.
- Operational data ownership, backup, update timing, and rollback policy.
- An approved long-term basemap provider and attribution policy.
- FMI policy compliance for CDN-loaded frontend libraries.
- A pinned base-image version or digest instead of `ubi9/python-312:latest`.
- Automated smoke tests and a small local fixture dataset.

## Source of Truth

Use this runbook and the manifests in `deploy/` for deployment. Use `server.py`, `config/app.json`, and `data/README.md` for the application data contract. For facts not represented in Git, query the live OpenShift resources before changing them.
