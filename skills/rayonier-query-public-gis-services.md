---
name: rayonier-query-public-gis-services
description: Discover and query Rayonier's anonymously readable ArcGIS REST services (fee ownership, hunting lease units, bee-lease sites, land-sale tracts, forest inventory, fire overlays) on gis.rayonier.com.
api: rayonier:rayonier-gis-services
generated: '2026-09-17'
method: generated
source: arcgis/rayonier-arcgis-service-inventory.json, arcgis/rayonier-arcgis-layer-schemas.json, conventions/rayonier-conventions.yml
operations:
  - GET /arcgis/rest/services/{folder}?f=json
  - GET /arcgis/rest/services/{service}/{type}/{layerId}?f=json
  - GET /arcgis/rest/services/{service}/{type}/{layerId}/query
---

# Query Rayonier's public GIS services

Rayonier publishes no developer documentation. Everything here was verified by calling the
surface anonymously on 2026-09-17. Treat the surface as READ-ONLY.

## 1. Discover a service

```
GET https://gis.rayonier.com/arcgis/rest/services/Public?f=json
GET https://gis.rayonier.com/arcgis/rest/services/Hosted?f=json
```

Each `services[]` entry is `{name, type}`; `type` is `FeatureServer` or `MapServer`. Hosted/
services are FeatureServer only — asking for a MapServer that does not exist returns
`{"error":{"code":404,"message":"Service not found"}}` inside an HTTP 200. Always append
`f=json`; the default is HTML.

## 2. Read the layer before querying it

```
GET https://gis.rayonier.com/arcgis/rest/services/Public/Rayonier_FEE_Ownership_Public/FeatureServer/0?f=json
```

Read `fields[]` (field names are case-sensitive in `where`), `maxRecordCount` (5000 here,
2000 on most layers, 1000 on some Hosted layers), `supportedQueryFormats` and
`geometryType`. A `where` on a field the layer lacks returns
`{"error":{"code":400,"message":"Unable to complete operation."}}` — again inside HTTP 200.

## 3. Size the call, then page it

```
GET .../FeatureServer/0/query?where=1%3D1&returnCountOnly=true&f=json      -> {"count":6216}
GET .../FeatureServer/0/query?where=1%3D1&outFields=ENTITYID,GISACRES&returnGeometry=false&resultOffset=0&resultRecordCount=2000&f=json
```

Page with `resultOffset` / `resultRecordCount` until `exceededTransferLimit` is absent or
false. Ask for `returnGeometry=false` unless you need shapes — a full fee-ownership pull with
geometry is ~25 MB. For GeoJSON use `f=geojson` and `outSR=4326` (the server default is Web
Mercator, 3857).

## 4. Marquee layers

| Need | Layer |
|---|---|
| Rayonier fee-owned land | `Public/Rayonier_FEE_Ownership_Public/FeatureServer/0` |
| Hunting lease units / permit areas / access points / food plots | `Public/Hunting_RLU_Permits_Salesforce/MapServer/{0,3,2,6}` |
| Active and recently completed harvests on hunting units | `Public/Hunting_TimberSales_SalesForce/MapServer/{0,1}` |
| Bee-lease sites (1 = Available, 0 = Unavailable) | `Public/BeeLease_WebSite/MapServer/{1,0}` |
| Land-sale tracts (Raydient Rural) | `Public/RaydientWebSite_TractPolygon/MapServer/0` |
| NIFC fire perimeters overlapping Rayonier land | `Public/FirePerimeters/FeatureServer/0` |
| Forest stand inventory, US South | `Hosted/INV_STAND_DESC_SOUTH/FeatureServer/0` |

## Rules

- No credential is needed on Public/ or Hosted/; the Utilities folder answers `499 Token Required` and there is no self-signup — stop there.
- Errors arrive in the body with HTTP 200. Check for an `error` key on every response.
- There are no rate-limit headers; be polite (the inventory pass above ran at ~3 req/s without incident).
- Nine services advertise Create/Update/Delete. Do not call applyEdits, addFeatures, updateFeatures or deleteFeatures — there is no idempotency key and no undo.
- Data freshness: hunting and bee-lease layers carry `AsOfDate`; several layers are rebuilt nightly by FME.
