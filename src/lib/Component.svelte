<svelte:options
  customElement={{ tag: "tiangongtracker-wgc", shadow: "open" }}
/>

<script>
  import { onMount, onDestroy } from "svelte";
  // import * as satellite from "satellite.js";
  import Map from "ol/Map";
  import View from "ol/View";
  import TileLayer from "ol/layer/Tile";
  import VectorLayer from "ol/layer/Vector";
  import VectorSource from "ol/source/Vector";
  import XYZ from "ol/source/XYZ";
  import Feature from "ol/Feature";
  import Point from "ol/geom/Point";
  import LineString from "ol/geom/LineString";
  import MultiLineString from "ol/geom/MultiLineString";
  import { fromLonLat } from "ol/proj";
  import { Style, Stroke, Circle, Fill, Text } from "ol/style";
  import GeoJSON from "ol/format/GeoJSON";

  let lat = 0;
  let lng = 0;
  let alt = 0;
  let velocity = 0;
  let currentTimeDisplay = "0000-00-00 00:00:00";
  let loading = true;
  let error = null;

  let mapContainer;
  let simContainer;

  let worldMap;
  let simMap;
  let stationFeature;
  let fullPathFeature;
  let highlightPathFeature;
  let vectorSource;
  let updateInterval;

  let tleLine1 =
    "1 48274U 21035A   26156.07366053  .00026328  00000-0  31558-3 0  9992";
  let tleLine2 =
    "2 48274  41.4694  35.6942 0009569 344.8796  15.1755 15.60275999291272";

  function loadSatelliteLibrary() {
    return new Promise((resolve, reject) => {
      if (window.satellite) {
        resolve();
        return;
      }
      const script = document.createElement("script");
      script.src =
        "https://cdnjs.cloudflare.com/ajax/libs/satellite.js/5.0.0/satellite.min.js";
      script.onload = () => resolve();
      script.onerror = () =>
        reject(new Error("Failed to load satellite engine from CDN"));
      document.head.appendChild(script);
    });
  }

  async function fetchLiveTLE() {
    try {
      const res = await fetch(
        "https://corsproxy.io/?https://celestrak.org/NORAD/elements/gp.php?CATNR=48274&FORMAT=tle",
      );
      if (res.ok) {
        const text = await res.text();
        const lines = text
          .split("\n")
          .map((l) => l.trim())
          .filter((l) => l.length > 0);

        if (lines.length >= 3 && lines[0].includes("CSS")) {
          tleLine1 = lines[1];
          tleLine2 = lines[2];
          error = null;
        }
      }
    } catch (e) {
      console.warn("Using verified 2026 baseline parameters:", e);
    } finally {
      loading = false;
      setTimeout(() => {
        initTracker();
      }, 100);
    }
  }

  function getSatellitePosition(time) {
    try {
      const satrec = satellite.twoline2satrec(tleLine1, tleLine2);
      const positionAndVelocity = satellite.propagate(satrec, time);
      const positionEci = positionAndVelocity.position;
      const velocityEci = positionAndVelocity.velocity;
      const gmst = satellite.gstime(time);

      if (positionEci) {
        const positionGd = satellite.eciToGeodetic(positionEci, gmst);
        let longitude = satellite.degreesLong(positionGd.longitude);
        let latitude = satellite.degreesLat(positionGd.latitude);
        const altitude = positionGd.height;

        if (longitude > 180) longitude -= 360;
        if (longitude < -180) longitude += 360;
        if (Math.abs(latitude) > 90) {
          latitude = latitude > 0 ? 180 - latitude : -180 - latitude;
        }

        const velMag =
          Math.sqrt(
            Math.pow(velocityEci.x, 2) +
              Math.pow(velocityEci.y, 2) +
              Math.pow(velocityEci.z, 2),
          ) * 3600;

        return { lng: longitude, lat: latitude, alt: altitude, vel: velMag };
      }
    } catch (err) {
      console.error("Math engine parsing exception:", err);
    }
    return null;
  }

  function generateFullOrbitPath() {
    const segments = [];
    let currentSegment = [];
    const now = new Date();

    for (let i = -48; i <= 48; i += 2) {
      const futureTime = new Date(now.getTime() + i * 60 * 1000);
      const pos = getSatellitePosition(futureTime);
      if (pos) {
        const coords = fromLonLat([pos.lng, pos.lat]);
        
        if (currentSegment.length > 0) {
          const prevPos = getSatellitePosition(new Date(now.getTime() + (i - 2) * 60 * 1000));
          if (prevPos && Math.abs(pos.lng - prevPos.lng) > 180) {
            segments.push(currentSegment);
            currentSegment = [];
          }
        }
        currentSegment.push(coords);
      }
    }
    if (currentSegment.length > 0) segments.push(currentSegment);
    return segments; 
  }

  function generateHighlightPath() {
    const segments = [];
    let currentSegment = [];
    const now = new Date();
    const step = 0.2;

    for (let i = -1; i <= 1; i += step) {
      const futureTime = new Date(now.getTime() + i * 60 * 1000);
      const pos = getSatellitePosition(futureTime);
      if (pos) {
        const coords = fromLonLat([pos.lng, pos.lat]);

        if (currentSegment.length > 0) {
          const prevPos = getSatellitePosition(new Date(now.getTime() + (i - step) * 60 * 1000));
          if (prevPos && Math.abs(pos.lng - prevPos.lng) > 180) {
            segments.push(currentSegment);
            currentSegment = [];
          }
        }
        currentSegment.push(coords);
      }
    }
    if (currentSegment.length > 0) segments.push(currentSegment);
    return segments;
  }

  function formatLocal24hTime(date) {
    const options = {
      year: 'numeric',
      month: '2-digit',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
      hour12: false 
    };
    const formatter = new Intl.DateTimeFormat(navigator.language || 'en-US', options);
    const parts = formatter.formatToParts(date);
    const p = {};
    parts.forEach(part => { p[part.type] = part.value; });
    
    return `${p.year}-${p.month}-${p.day} ${p.hour}:${p.minute}:${p.second}`;
  }

  function initTracker() {
    if (!mapContainer || !simContainer) return;

    const currentPos = getSatellitePosition(new Date());
    if (!currentPos) {
      error = "Orbital matrix initialization error.";
      return;
    }

    lat = currentPos.lat;
    lng = currentPos.lng;
    alt = currentPos.alt;
    velocity = currentPos.vel;
    currentTimeDisplay = formatLocal24hTime(new Date());

    const initialCoords = fromLonLat([lng, lat]);

    stationFeature = new Feature({ geometry: new Point(initialCoords) });
    stationFeature.setStyle(
      new Style({
        image: new Circle({
          radius: 5,
          fill: new Fill({ color: "#00ffcc" }),
          stroke: new Stroke({ color: "#ffffff", width: 1 }),
        }),
      }),
    );

    fullPathFeature = new Feature({
      geometry: new MultiLineString(generateFullOrbitPath()),
    });
    fullPathFeature.setStyle(
      new Style({
        stroke: new Stroke({
          color: "rgba(255, 255, 255, 0.6)",
          width: 1,
          lineDash: [1, 2],
        }),
      }),
    );

    highlightPathFeature = new Feature({
      geometry: new MultiLineString(generateHighlightPath()),
    });
    highlightPathFeature.setStyle(
      new Style({
        stroke: new Stroke({ color: "#ffffff", width: 1 }),
      }),
    );

    vectorSource = new VectorSource({
      features: [fullPathFeature, highlightPathFeature, stationFeature],
    });

    const transparentBaseWorldLayer = new VectorLayer({
      source: new VectorSource({
        url: "https://raw.githubusercontent.com/datasets/geo-boundaries-world-110m/master/countries.geojson",
        format: new GeoJSON(),
      }),
      declutter: true, 
      style: function (feature) {
        const styles = [];
        styles.push(
          new Style({
            text: new Text({
              font: '100 10px "Barlow Semi Condensed", sans-serif',
              text: feature.get("name") || "",
              fill: new Fill({ color: "rgba(255, 255, 255, 0.7)" }),
              placement: 'point', 
              overflow: false, 
              textAlign: 'center',
              textBaseline: 'middle'
            }),
          })
        );

        const geometry = feature.getGeometry();
        if (geometry) {
          styles.push(
            new Style({
              geometry: function (f) {
                const geom = f.getGeometry();
                if (geom.getType() === 'Polygon') {
                  return new LineString(geom.getCoordinates()[0]);
                } else if (geom.getType() === 'MultiPolygon') {
                  const lineStrings = geom.getPolygons().map(p => new LineString(p.getCoordinates()[0]));
                  return new MultiLineString(lineStrings.map(l => l.getCoordinates()));
                }
                return geom;
              },
              stroke: new Stroke({
                color: "rgba(255, 255, 255, 0.35)", 
                width: 1,                           
              }),
            })
          );
        }

        return styles;
      },
    });

    const placesLayer = new VectorLayer({
      source: new VectorSource({
        url: "https://raw.githubusercontent.com/nvkelso/natural-earth-vector/master/geojson/ne_110m_populated_places.geojson",
        format: new GeoJSON(),
      }),
      declutter: true, 
      style: function (feature) {
        return new Style({
          image: new Circle({
            radius: 2,
            fill: new Fill({ color: "rgba(255, 255, 255, 0.4)" }),
          }),
          text: new Text({
            font: '100 10px "Barlow Semi Condensed", sans-serif',
            text: feature.get("NAME") || "", 
            fill: new Fill({ color: "rgba(255, 255, 255, 0.7)" }),
            offsetX: 8, 
            textAlign: 'left',
            textBaseline: 'middle'
          }),
        });
      },
    });

    worldMap = new Map({
      target: mapContainer,
      controls: [],
      layers: [
        transparentBaseWorldLayer,
        placesLayer,
        new VectorLayer({ source: vectorSource }),
      ],
      view: new View({ center: initialCoords, zoom: 2, minZoom: 1.5 }),
    });

    simMap = new Map({
      target: simContainer,
      controls: [],
      layers: [
        new TileLayer({
          source: new XYZ({
            url: "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
          }),
        }),
      ],
      view: new View({ center: initialCoords, zoom: 5.5, rotation: 0 }),
    });

    setTimeout(() => {
      worldMap.updateSize();
      simMap.updateSize();
      vectorSource.changed();
    }, 200);

    updateInterval = setInterval(() => {
      const now = new Date();
      currentTimeDisplay = formatLocal24hTime(now);

      const pos = getSatellitePosition(now);
      if (pos) {
        lat = pos.lat;
        lng = pos.lng;
        alt = pos.alt;
        velocity = pos.vel;

        const updatedCoords = fromLonLat([lng, lat]);

        stationFeature.getGeometry().setCoordinates(updatedCoords);
        fullPathFeature.getGeometry().setCoordinates(generateFullOrbitPath());
        highlightPathFeature
          .getGeometry()
          .setCoordinates(generateHighlightPath());

        vectorSource.changed();

        worldMap.getView().setCenter(updatedCoords);
        simMap.getView().setCenter(updatedCoords);
        simMap.getView().setRotation(simMap.getView().getRotation() + 0.002);
      }
    }, 1000);
  }

  onMount(async () => {
    try {
      await loadSatelliteLibrary();
      await fetchLiveTLE();
    } catch (err) {
      error = err.message;
      loading = false;
    }
  });

  onDestroy(() => {
    if (updateInterval) clearInterval(updateInterval);
  });
</script>

<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/ol@8.2.0/ol.css"
  type="text/css"
/>

<section class="trackerCtn">
  {#if loading}
    <div class="overlay">Loading live orbital tracker......</div>
  {:else if error}
    <div class="overlay error">{error}</div>
  {/if}

  <div class="mapCtn">
    <div class="panelHead">Tiangong orbital tracker</div>
    <div class="livingDataBox">
      <div>Real time: {currentTimeDisplay}</div>
      <div>Latitude: {lat.toFixed(4)}°</div>
      <div>Longitude: {lng.toFixed(4)}°</div>
      <div>Altitude: {alt.toFixed(1)} km</div>
      <div>Velocity: {Math.round(velocity).toLocaleString()} km/h</div>
    </div>
    <div class="mapPanel">
      <div class="locationMap">
        <div class="note">Ground Track View</div>
        <div bind:this={mapContainer} class="map"></div>
      </div>
      <div class="livingCameraBox">
        <div class="note">Simulated earth view (Nadir Camera)</div>
        <div class="crosshairWrapper">
          <div class="crosshair"></div>
          <div bind:this={simContainer} class="radarMap" style="width: 100%; height: 100%;"></div>
      </div>
    </div>
    </div>
  </div>
</section>

<style>
  :global(html),
  :global(body) {
    overflow-x: hidden;
    overflow-y: auto !important;
    margin: 0;
    padding: 0;
    min-width: 290px;
    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
    -webkit-font-smoothing: antialiased;
    font-family: "Barlow Semi Condensed", sans-serif;
  }

  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: "Barlow Semi Condensed", sans-serif;
  }

  .trackerCtn, .mapCtn{
    background-color: #000000;
  }

  .mapPanel{
    width: 100%;
    max-width: 1200px;
    display: flex;
    gap: 20px;
    flex-direction: row;
    flex-wrap: nowrap;
    align-content: center;
    justify-content: flex-start;
    align-items: center;
  }

  .locationMap, .livingCameraBox{
    width: 49%;
    height: 40vh;
    max-height: 600px;
    pointer-events: none;
    position: relative;
    border: 1px solid #ffffff30;
    border-radius: 10px;
    margin-bottom: 10px;
  }

  .map{
    width: 100%;
    height: 100%;
    background: transparent !important;
  }

  .panelHead {
    color: #ffffff;
    font-size: 15px;
    font-weight: 500;
    margin: 10px;
  }

  .livingDataBox {
    font-size: 12px;
    font-weight: 300;
    color: #00ffcc;
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    align-content: center;
    justify-content: flex-start;
    align-items: center;
    gap:30px;
    margin: 10px;
  }

  .livingCameraBox .note, .locationMap .note {
    position: absolute;
    bottom: 10px;
    right: 10px;
    font-size: 12px;
    font-weight: 500;
    z-index: 10;
    padding: 2px 4px;
    background-color: #ffffff;
  }

  .crosshairWrapper {
    position: relative;
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  .crosshair {
    position: absolute;
    top: 50%;
    left: 50%;
    width: 30px;
    height: 30px;
    transform: translate(-50%, -50%);
    border: 2px solid rgba(255, 68, 68, 0.924);
    border-radius: 50%;
    z-index: 10;
    pointer-events: none;
  }
  .crosshair::before,
  .crosshair::after {
    content: "";
    position: absolute;
    background: rgba(255, 68, 68, 0.924);
  }
  .crosshair::before {
    top: 50%;
    left: -8px;
    width: 46px;
    height: 1px;
    transform: translateY(-50%);
  }
  .crosshair::after {
    left: 50%;
    top: -8px;
    height: 46px;
    width: 1px;
    transform: translateX(-50%);
  }

  .overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(13, 15, 18, 0.95);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 10;
    font-size: 14px;
    color: #00ffcc;
    text-align: center;
  }
  .error {
    color: #ff4444;
  }

  @media screen and (max-width: 600px){
    .mapPanel{
      display: unset;
    }

    .locationMap, .livingCameraBox{
      width: 98%;
      height: 30vh;
      margin: 10px auto;
    }
    .livingDataBox{
      flex-wrap: wrap;
      gap: 5px;
    }
    .livingDataBox div{
      min-width: 160px;
    }
  }
</style>
