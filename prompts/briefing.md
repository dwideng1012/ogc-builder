import React, { useEffect, useMemo, useRef, useState } from "react";

/**
 * OGC Request Builder – stable rewrite + clipboard fallback fix
 *
 * Fixes
 * - Clipboard API NotAllowedError: now uses a safe fallback (document.execCommand) and manual select button.
 * - Adds visible copy status message, plus a "Vali URL" helper to select the URL for manual copy.
 *
 * Features (unchanged)
 * - WMS (1.3.0 / 1.1.1 / 1.0.0): GetMap, GetFeatureInfo, GetLegendGraphic, GetCapabilities
 * - WFS (2.0.0 / 1.1.0 / 1.0.0): GetFeature, DescribeFeatureType, GetCapabilities
 * - OGC API Features (OAF): /collections, /collections/{id}/items, /conformance
 * - EPSG:3301-only BBOX presets + WMS 1.3.0 axis-order variant helper
 * - OUTPUTFORMAT options for WFS parsed from Capabilities
 * - Namespace/prefix (nt "ms:") eemaldus LAYERS/TYPENAMES väärtustest
 * - URL eelvaade (auto + käsitsi uuendus), ajalugu, presetid
 * - Self-testid (ilma võrguta) regressioonide vastu
 */

// ---------------------- Utility helpers ----------------------
function deriveOafRoot(input: string) {
  const url = (input || "").split("?")[0].replace(/\/?$/, "");
  // .../ows/{serviceName}  ->  .../{serviceName}/ogcapi
  const m = url.match(/^(.*)\/ows\/([^\/?#]+)$/);
  if (m) return `${m[1]}/${m[2]}/ogcapi`;
  // already .../ogcapi[/...] -> clamp to .../ogcapi
  const m2 = url.match(/^(.*)\/ogcapi(?:\/.*)?$/);
  if (m2) return `${m2[1]}/ogcapi`;
  return url;
}

const PRESET_KEY = "ogc_presets_v1";
const URL_HISTORY_KEY = "ogc_url_history_v1";

const BBOX_PRESETS_3301: Record<string, string> = {
  "Terve Eesti": "40500,5993000,1064500,7027400",
  "Ida-Virumaa": "705000,6589000,876000,7080000",
  "Hiiumaa": "307000,6535000,415000,6630000",
  "Tallinn": "515000,6588000,560000,6620000",
  "Lihula": "424000,6478000,448000,6504000",
};

function loadJson<T>(key: string, fallback: T): T {
  try {
    const raw = localStorage.getItem(key);
    return raw ? (JSON.parse(raw) as T) : fallback;
  } catch {
    return fallback;
  }
}
function saveJson(key: string, value: any) {
  localStorage.setItem(key, JSON.stringify(value));
}

function kvToQuery(params: Record<string, string | number | boolean | undefined>) {
  const usp = new URLSearchParams();
  for (const [k, v] of Object.entries(params)) {
    if (v === undefined || v === null || v === "") continue;
    usp.append(k, String(v));
  }
  return usp.toString();
}
function ensureNoTrailingQ(url: string) {
  return url.endsWith("?") ? url.slice(0, -1) : url;
}
function mergeUrl(base: string, query: string) {
  const sep = base.includes("?") ? (base.endsWith("?") || base.endsWith("&") ? "" : "&") : "?";
  return `${base}${sep}${query}`;
}
function stripNs(name: string) {
  const i = name.indexOf(":");
  return i >= 0 ? name.slice(i + 1) : name;
}
function parseXml(xmlText: string) {
  return new DOMParser().parseFromString(xmlText, "application/xml");
}
function textContent(el: Element | null): string {
  return el ? (el.textContent || "").trim() : "";
}
function getAll(el: Element | Document, tag: string): Element[] {
  return Array.from(el.getElementsByTagName(tag));
}

function parseWmsLayers(xmlText: string) {
  const xml = parseXml(xmlText);
  return getAll(xml, "Layer")
    .filter((el) => el.querySelector(":scope > Name"))
    .map((el) => ({
      name: textContent(el.querySelector(":scope > Name")),
      title: textContent(el.querySelector(":scope > Title")),
    }))
    .filter((l) => l.name);
}
function parseWfsFeatureTypes(xmlText: string) {
  const xml = parseXml(xmlText);
  return getAll(xml, "FeatureType")
    .map((el) => ({ name: textContent(el.querySelector("Name")), title: textContent(el.querySelector("Title")) }))
    .filter((l) => l.name);
}
function parseWfsOutputFormats(xmlText: string) {
  try {
    const xml = parseXml(xmlText);
    const ops = Array.from(xml.getElementsByTagName("Operation")).filter((o) => (o.getAttribute("name") || "").toLowerCase() === "getfeature");
    const out: string[] = [];
    ops.forEach((op) => {
      Array.from(op.getElementsByTagName("Parameter")).forEach((p) => {
        if ((p.getAttribute("name") || "").toLowerCase() === "outputformat") {
          Array.from(p.getElementsByTagName("Value")).forEach((v) => {
            const t = (v.textContent || "").trim();
            if (t && !out.includes(t)) out.push(t);
          });
        }
      });
    });
    return out;
  } catch {
    return [];
  }
}

// Swap axis order helper: minx,miny,maxx,maxy -> miny,minx,maxy,maxx
function swapBboxOrder(b: string) {
  const p = b.split(",").map((s) => s.trim());
  if (p.length !== 4) return b;
  return `${p[1]},${p[0]},${p[3]},${p[2]}`;
}

// ---------------------- Small subcomponent ----------------------
function BboxVariants({ bbox, onApply }: { bbox: string; onApply: (v: string) => void }) {
  const for111 = bbox; // minx,miny,maxx,maxy
  const for130 = swapBboxOrder(bbox); // teljevahetus (näidik WMS 1.3.0 jaoks)
  return (
    <div className="grid grid-cols-2 gap-2 mt-2">
      <div>
        <label className="text-xs text-gray-600">WMS 1.1.1 / 1.0.0 (SRS)</label>
        <div className="flex gap-2 items-center">
          <input className="w-full border rounded-xl px-2 py-1 font-mono text-xs" value={for111} readOnly />
          <button className="px-2 py-1 border rounded-lg text-xs" onClick={() => onApply(for111)}>Kasuta</button>
        </div>
      </div>
      <div>
        <label className="text-xs text-gray-600">WMS 1.3.0 (CRS)</label>
        <div className="flex gap-2 items-center">
          <input className="w-full border rounded-xl px-2 py-1 font-mono text-xs" value={for130} readOnly />
          <button className="px-2 py-1 border rounded-lg text-xs" onClick={() => onApply(for130)}>Kasuta</button>
        </div>
      </div>
    </div>
  );
}

// ---------------------- Main Component ----------------------
export default function OGCRequestBuilder() {
  // Core state
  const [baseUrl, setBaseUrl] = useState("https://teenus.maaamet.ee/keskkonnatervis/ogcapi/");
  const [service, setService] = useState<"WMS" | "WFS" | "OAF">("WMS");

  const [wmsVersion, setWmsVersion] = useState("1.3.0"); // WMS: 1.3.0 / 1.1.1 / 1.0.0
  const [wfsVersion, setWfsVersion] = useState("2.0.0"); // WFS: 2.0.0 / 1.1.0 / 1.0.0

  const [request, setRequest] = useState<string>("GetMap");

  const [layers, setLayers] = useState<{ name: string; title: string }[]>([]);
  const [selectedLayers, setSelectedLayers] = useState<string[]>([]);

  const [collections, setCollections] = useState<{ id: string; title?: string }[]>([]);
  const [selectedCollection, setSelectedCollection] = useState<string>("");

  // Parameters
  const [crs] = useState("EPSG:3301"); // fixed as requested
  const [bbox, setBbox] = useState(BBOX_PRESETS_3301["Terve Eesti"]);
  const [selectedBboxPreset, setSelectedBboxPreset] = useState<string>("Terve Eesti");

  const [width, setWidth] = useState(1280);
  const [height, setHeight] = useState(1000);
  const [format, setFormat] = useState("image/png");
  const [transparent, setTransparent] = useState(true);
  const [styles, setStyles] = useState("default");

  const [customParams, setCustomParams] = useState<{ key: string; value: string }[]>([
    { key: "kogum_id", value: "1063800_1" },
  ]);

  // OAF params
  const [oafLimit, setOafLimit] = useState(100);
  const [oafOffset, setOafOffset] = useState(0);
  const [oafDatetime, setOafDatetime] = useState("");
  const [oafFilter, setOafFilter] = useState("");

  // WFS formats
  const [wfsFormats, setWfsFormats] = useState<string[]>([]);
  const [wfsOutputFormat, setWfsOutputFormat] = useState<string>("\n");

  // History + presets
  const [urlHistory, setUrlHistory] = useState<string[]>(() => loadJson<string[]>(URL_HISTORY_KEY, []));
  const [presets, setPresets] = useState<Record<string, any>>(() => loadJson(PRESET_KEY, {}));
  const [newPresetName, setNewPresetName] = useState("");

  // Output
  const [urlPreview, setUrlPreview] = useState("");
  const [autoUpdate, setAutoUpdate] = useState(true);
  const [loading, setLoading] = useState(false);
  const [responseText, setResponseText] = useState<string | null>(null);
  const [responseBlobUrl, setResponseBlobUrl] = useState<string | null>(null);
  const [error, setError] = useState<string | null>(null);

  // Clipboard helper state
  const urlAreaRef = useRef<HTMLTextAreaElement | null>(null);
  const [copyInfo, setCopyInfo] = useState<string>("");

  const derivedOafRoot = useMemo(() => deriveOafRoot(baseUrl), [baseUrl]);

  // --------------- Effects ---------------
  // Reset request when switching service to a sensible default
  useEffect(() => {
    setSelectedLayers([]);
    setSelectedCollection("");
    if (service === "WMS") setRequest("GetMap");
    if (service === "WFS") setRequest("GetFeature");
    if (service === "OAF") setRequest("collections");
  }, [service]);

  // Auto-load capabilities/collections when service/version/baseUrl changes
  useEffect(() => {
    fetchCatalog();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [service, wmsVersion, wfsVersion, baseUrl]);

  // Update URL preview automatically
  useEffect(() => {
    if (autoUpdate) setUrlPreview(buildUrl());
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [autoUpdate, baseUrl, service, wmsVersion, wfsVersion, request, selectedLayers, selectedCollection, styles, bbox, width, height, format, transparent, customParams, oafLimit, oafOffset, oafDatetime, oafFilter]);

  // --------------- Builders ---------------
  function extrasFromCustom(): Record<string, string> {
    const out: Record<string, string> = {};
    for (const { key, value } of customParams) {
      if (!key || value === undefined || value === null || value === "") continue;
      out[key] = value;
    }
    return out;
  }

  function buildUrl(): string {
    const base = ensureNoTrailingQ(baseUrl.trim());
    if (!base) return "";
    const extras = extrasFromCustom();

    if (service === "WMS") {
      // Param name: SRS for 1.1.x/1.0.0, CRS for 1.3.0
      const useCRS = wmsVersion === "1.3.0";
      const layerList = selectedLayers.map(stripNs).join(",");
      const stylesParam = styles === "" ? "default" : styles;

      if (request === "GetCapabilities") {
        return mergeUrl(base, kvToQuery({ SERVICE: "WMS", REQUEST: "GetCapabilities", VERSION: wmsVersion, ...extras }));
      }

      const common: Record<string, any> = { SERVICE: "WMS", VERSION: wmsVersion, [useCRS ? "CRS" : "SRS"]: crs };

      if (request === "GetMap") {
        const q = kvToQuery({
          ...common,
          REQUEST: "GetMap",
          LAYERS: layerList,
          STYLES: stylesParam,
          BBOX: bbox,
          WIDTH: width,
          HEIGHT: height,
          FORMAT: format,
          TRANSPARENT: transparent ? "TRUE" : "FALSE",
          ...extras,
        });
        return mergeUrl(base, q);
      }

      if (request === "GetFeatureInfo") {
        const clickXParam = useCRS ? "I" : "X";
        const clickYParam = useCRS ? "J" : "Y";
        const q = kvToQuery({
          ...common,
          REQUEST: "GetFeatureInfo",
          LAYERS: layerList,
          QUERY_LAYERS: layerList,
          STYLES: stylesParam,
          BBOX: bbox,
          WIDTH: width,
          HEIGHT: height,
          INFO_FORMAT: "application/json",
          [clickXParam]: Math.floor(width / 2),
          [clickYParam]: Math.floor(height / 2),
          ...extras,
        });
        return mergeUrl(base, q);
      }

      if (request === "GetLegendGraphic") {
        const q = kvToQuery({ SERVICE: "WMS", VERSION: wmsVersion, REQUEST: "GetLegendGraphic", LAYER: stripNs(selectedLayers[0] || ""), FORMAT: "image/png", SCALE: 10000, ...extras });
        return mergeUrl(base, q);
      }

      return "";
    }

    if (service === "WFS") {
      if (request === "GetCapabilities") {
        return mergeUrl(base, kvToQuery({ SERVICE: "WFS", REQUEST: "GetCapabilities", VERSION: wfsVersion, ...extras }));
      }
      const typeKey = wfsVersion === "2.0.0" ? "TYPENAMES" : "TYPENAME";
      const layerList = selectedLayers.map(stripNs).join(",");

      if (request === "GetFeature") {
        const params: Record<string, any> = { SERVICE: "WFS", REQUEST: "GetFeature", VERSION: wfsVersion, [typeKey]: layerList, SRSNAME: crs, COUNT: 100, STARTINDEX: 0, ...extras };
        if (wfsOutputFormat && wfsOutputFormat.trim()) params["OUTPUTFORMAT"] = wfsOutputFormat;
        return mergeUrl(base, kvToQuery(params));
      }

      if (request === "DescribeFeatureType") {
        const params: Record<string, any> = { SERVICE: "WFS", REQUEST: "DescribeFeatureType", VERSION: wfsVersion, ...extras };
        if (selectedLayers.length > 0) params[typeKey] = stripNs(selectedLayers[0]);
        return mergeUrl(base, kvToQuery(params));
      }
      return "";
    }

    // OAF
    const root = deriveOafRoot(base);
    if (request === "conformance") {
      return `${root}/conformance`;
    }
    if (request === "collections") {
      const q = kvToQuery({ limit: oafLimit, ...extras });
      return `${root}/collections${q ? `?${q}` : ""}`;
    }
    if (request === "items") {
      const id = selectedCollection;
      if (!id) return `${root}/collections`;
      const q = kvToQuery({ f: "geojson", bbox, "bbox-crs": `http://www.opengis.net/def/crs/EPSG/0/3301`, crs: `http://www.opengis.net/def/crs/EPSG/0/3301`, limit: oafLimit, offset: oafOffset, datetime: oafDatetime, filter: oafFilter, ...extras });
      return `${root}/collections/${encodeURIComponent(id)}/items?${q}`;
    }
    return "";
  }

  // --------------- IO ---------------
  async function fetchCatalog() {
    try {
      setError(null);
      setLayers([]);
      setCollections([]);
      if (baseUrl && !urlHistory.includes(baseUrl)) {
        const next = [baseUrl, ...urlHistory].slice(0, 50);
        setUrlHistory(next);
        saveJson(URL_HISTORY_KEY, next);
      }
      if (service === "WMS") {
        const url = mergeUrl(ensureNoTrailingQ(baseUrl), kvToQuery({ SERVICE: "WMS", REQUEST: "GetCapabilities", VERSION: wmsVersion }));
        const res = await fetch(url);
        const txt = await res.text();
        setLayers(parseWmsLayers(txt));
      } else if (service === "WFS") {
        const url = mergeUrl(ensureNoTrailingQ(baseUrl), kvToQuery({ SERVICE: "WFS", REQUEST: "GetCapabilities", VERSION: wfsVersion }));
        const res = await fetch(url);
        const txt = await res.text();
        setLayers(parseWfsFeatureTypes(txt));
        setWfsFormats(parseWfsOutputFormats(txt));
      } else if (service === "OAF") {
        const root = deriveOafRoot(baseUrl);
        const url = `${root}/collections`;
        const res = await fetch(url, { headers: { Accept: "application/json" } });
        if (!res.ok) throw new Error(`Collections request failed (${res.status})`);
        const json = await res.json();
        const cols = (json.collections || []).map((c: any) => ({ id: c.id, title: c.title }));
        setCollections(cols);
      }
    } catch (e: any) {
      setError(e?.message || String(e));
    }
  }

  async function doRequest() {
    try {
      setLoading(true);
      setError(null);
      setResponseText(null);
      setResponseBlobUrl(null);
      const url = urlPreview || buildUrl();
      if (!url) throw new Error("URL puudub");
      const res = await fetch(url);
      const ct = res.headers.get("content-type") || "";
      if (ct.startsWith("image")) {
        const blob = await res.blob();
        const u = URL.createObjectURL(blob);
        setResponseBlobUrl(u);
      } else {
        const txt = await res.text();
        setResponseText(txt);
      }
    } catch (e: any) {
      setError(e?.message || String(e));
    } finally {
      setLoading(false);
    }
  }

  // --------------- Clipboard helpers ---------------
  async function copyUrlToClipboard() {
    const text = urlPreview;
    if (!text) {
      setCopyInfo("URL puudub");
      setTimeout(() => setCopyInfo(""), 2500);
      return;
    }
    // Try modern Clipboard API first
    try {
      await navigator.clipboard.writeText(text);
      setCopyInfo("Kopeeritud lõikepuhvrisse");
    } catch (err) {
      // Fallback using a temporary textarea and execCommand('copy')
      try {
        const ta = document.createElement("textarea");
        ta.value = text;
        ta.setAttribute("readonly", "");
        ta.style.position = "fixed";
        ta.style.left = "-9999px";
        document.body.appendChild(ta);
        ta.focus();
        ta.select();
        const ok = document.execCommand("copy");
        document.body.removeChild(ta);
        if (ok) setCopyInfo("Kopeeritud (fallback)");
        else {
          setCopyInfo("Kopeerimine ebaõnnestus – vali käsitsi");
          if (urlAreaRef.current) {
            urlAreaRef.current.focus();
            urlAreaRef.current.select();
          }
        }
      } catch {
        setCopyInfo("Kopeerimine ebaõnnestus – vali käsitsi");
        if (urlAreaRef.current) {
          urlAreaRef.current.focus();
          urlAreaRef.current.select();
        }
      }
    } finally {
      setTimeout(() => setCopyInfo(""), 3000);
    }
  }
  function selectUrlForManualCopy() {
    if (urlAreaRef.current) {
      urlAreaRef.current.focus();
      urlAreaRef.current.select();
    }
  }

  // --------------- Presets ---------------
  function savePreset() {
    if (!newPresetName.trim()) return;
    const next = {
      ...presets,
      [newPresetName.trim()]: {
        baseUrl,
        service,
        wmsVersion,
        wfsVersion,
        request,
        selectedLayers,
        selectedCollection,
        crs,
        bbox,
        width,
        height,
        format,
        transparent,
        styles,
        customParams,
        oafLimit,
        oafOffset,
        oafDatetime,
        oafFilter,
        wfsOutputFormat,
      },
    };
    setPresets(next);
    saveJson(PRESET_KEY, next);
    setNewPresetName("");
  }
  function loadPreset(name: string) {
    const p = presets[name];
    if (!p) return;
    setBaseUrl(p.baseUrl);
    setService(p.service);
    setWmsVersion(p.wmsVersion);
    setWfsVersion(p.wfsVersion);
    setRequest(p.request);
    setSelectedLayers(p.selectedLayers || []);
    setSelectedCollection(p.selectedCollection || "");
    // crs is fixed (3301)
    setBbox(p.bbox || BBOX_PRESETS_3301["Terve Eesti"]);
    setWidth(p.width || 1280);
    setHeight(p.height || 1000);
    setFormat(p.format || "image/png");
    setTransparent(!!p.transparent);
    setStyles(p.styles || "default");
    setCustomParams(p.customParams || []);
    setOafLimit(p.oafLimit || 100);
    setOafOffset(p.oafOffset || 0);
    setOafDatetime(p.oafDatetime || "");
    setOafFilter(p.oafFilter || "");
    setWfsOutputFormat(p.wfsOutputFormat || "");
  }

  // --------------- Self-tests ---------------
  const [selfTest, setSelfTest] = useState<string>("");
  useEffect(() => {
    try {
      // WMS 1.1.1 GetMap SRS param
      const q1 = kvToQuery({ SERVICE: "WMS", REQUEST: "GetMap", VERSION: "1.1.1", SRS: "EPSG:3301", LAYERS: "a", STYLES: "default", BBOX: "1,2,3,4", WIDTH: 10, HEIGHT: 10, FORMAT: "image/png", TRANSPARENT: "TRUE" });
      if (!q1.includes("SRS=EPSG%3A3301")) throw new Error("WMS 1.1.1 must use SRS");
      // WMS 1.3.0 GetMap CRS param
      const q2 = kvToQuery({ SERVICE: "WMS", REQUEST: "GetMap", VERSION: "1.3.0", CRS: "EPSG:3301", LAYERS: "a", STYLES: "default", BBOX: "1,2,3,4", WIDTH: 10, HEIGHT: 10, FORMAT: "image/png", TRANSPARENT: "TRUE" });
      if (!q2.includes("CRS=EPSG%3A3301")) throw new Error("WMS 1.3.0 must use CRS");
      // WFS 2.0.0 TYPENAMES
      const q3 = kvToQuery({ SERVICE: "WFS", REQUEST: "GetFeature", VERSION: "2.0.0", TYPENAMES: "x", SRSNAME: "EPSG:3301" });
      if (!q3.includes("TYPENAMES=x")) throw new Error("WFS 2.0.0 must use TYPENAMES");
      // NEW: WFS 1.1.0 TYPENAME key check
      const q3b = kvToQuery({ SERVICE: "WFS", REQUEST: "GetFeature", VERSION: "1.1.0", TYPENAME: "x", SRSNAME: "EPSG:3301" });
      if (!q3b.includes("TYPENAME=x")) throw new Error("WFS 1.1.0 must use TYPENAME");
      // OAF items includes bbox-crs/crs 3301
      const q4 = kvToQuery({ bbox: "1,2,3,4", "bbox-crs": "http://www.opengis.net/def/crs/EPSG/0/3301", crs: "http://www.opengis.net/def/crs/EPSG/0/3301" });
      if (!q4.includes("EPSG/0/3301")) throw new Error("OAF must include EPSG:3301 links");
      // deriveOafRoot tests
      const r1 = deriveOafRoot("https://x/ows/keskkonnatervis");
      if (r1 !== "https://x/keskkonnatervis/ogcapi") throw new Error("deriveOafRoot ows→ogcapi failed");
      const r2 = deriveOafRoot("https://x/keskkonnatervis/ogcapi/?q=1");
      if (r2 !== "https://x/keskkonnatervis/ogcapi") throw new Error("deriveOafRoot clamp failed");
      // stripNs removes ms:
      if (stripNs("ms:kiht") !== "kiht") throw new Error("stripNs must remove namespace");
      // NEW: swapBboxOrder sanity
      const sw = swapBboxOrder("1,2,3,4");
      if (sw !== "2,1,4,3") throw new Error("swapBboxOrder failed");
      setSelfTest("OK: self-tests passed");
    } catch (e: any) {
      setSelfTest("FAIL: " + e.message);
    }
  }, []);

  // ---------------------- Render ----------------------
  const isWMS = service === "WMS";
  const isWFS = service === "WFS";
  const isOAF = service === "OAF";

  return (
    <div className="p-4 space-y-6 min-h-screen bg-white text-neutral-900 dark:bg-neutral-900 dark:text-neutral-100">
      {/* Top controls */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
          <h2 className="text-lg font-semibold">Teenuse alus-URL</h2>
          <input className="w-full border rounded-xl px-3 py-2 mt-2" value={baseUrl} onChange={(e) => setBaseUrl(e.target.value)} />
          <p className="text-sm text-gray-500 dark:text-gray-400 mt-1">Näiteks MapServer/GeoServer WMS/WFS OWS endpoint või OGC API root.</p>
          {isOAF && (
            <p className="text-xs text-gray-500 dark:text-gray-400 mt-1">Tõlgendatud OAF juur: <code className="font-mono">{derivedOafRoot}</code></p>
          )}
          {urlHistory.length > 0 && (
            <div className="mt-2">
              <div className="text-sm font-medium mb-1">Hiljutised alus-URLid</div>
              <div className="flex flex-wrap gap-2">
                {urlHistory.map((u) => (
                  <span key={u} className="inline-flex items-center gap-2 px-2 py-1 rounded-full border text-xs">
                    <button className="underline" onClick={() => setBaseUrl(u)} title={u}>{u.length > 50 ? u.slice(0, 50) + "…" : u}</button>
                    <button onClick={() => { const next = urlHistory.filter((x) => x !== u); setUrlHistory(next); saveJson(URL_HISTORY_KEY, next); }} title="kustuta">✕</button>
                  </span>
                ))}
              </div>
            </div>
          )}
        </div>

        <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
          <h2 className="text-lg font-semibold">Teenuse tüüp ja versioon</h2>
          <div className="flex gap-2 mt-2">
            <select className="border rounded-xl px-3 py-2" value={service} onChange={(e) => setService(e.target.value as any)}>
              <option>WMS</option>
              <option>WFS</option>
              <option>OAF</option>
            </select>
            {isWMS && (
              <select className="border rounded-xl px-3 py-2" value={wmsVersion} onChange={(e) => setWmsVersion(e.target.value)}>
                <option>1.3.0</option>
                <option>1.1.1</option>
                <option>1.0.0</option>
              </select>
            )}
            {isWFS && (
              <select className="border rounded-xl px-3 py-2" value={wfsVersion} onChange={(e) => setWfsVersion(e.target.value)}>
                <option>2.0.0</option>
                <option>1.1.0</option>
                <option>1.0.0</option>
              </select>
            )}
            {isOAF && (<span className="text-sm text-gray-500 dark:text-gray-400 px-2 py-2">OGC API Features</span>)}
            <button className="ml-auto px-3 py-2 rounded-xl border" onClick={fetchCatalog}>Laadi kihid/kollektsioonid</button>
          </div>

          {/* Request picker */}
          <div className="mt-3 flex gap-2">
            {isWMS && (
              <select className="border rounded-xl px-3 py-2" value={request} onChange={(e) => setRequest(e.target.value)}>
                <option>GetMap</option>
                <option>GetFeatureInfo</option>
                <option>GetLegendGraphic</option>
                <option>GetCapabilities</option>
              </select>
            )}
            {isWFS && (
              <select className="border rounded-xl px-3 py-2" value={request} onChange={(e) => setRequest(e.target.value)}>
                <option>GetFeature</option>
                <option>DescribeFeatureType</option>
                <option>GetCapabilities</option>
              </select>
            )}
            {isOAF && (
              <select className="border rounded-xl px-3 py-2" value={request} onChange={(e) => setRequest(e.target.value)}>
                <option>collections</option>
                <option>items</option>
                <option>conformance</option>
              </select>
            )}
          </div>
        </div>
      </div>

      {/* Layer/Collection picker */}
      <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
        <h2 className="text-lg font-semibold">Kihi/kollektsiooni valik</h2>
        {isWMS || isWFS ? (
          <div className="mt-2 flex flex-wrap gap-2">
            {layers.length === 0 && <div className="text-sm text-gray-500 dark:text-gray-400">— Lae Capabilities, et näha kihte —</div>}
            {layers.map((l) => {
              const checked = selectedLayers.includes(l.name);
              return (
                <label key={l.name} className={`px-2 py-1 border rounded-full text-sm cursor-pointer ${checked ? "bg-blue-600 text-white dark:bg-blue-500" : ""}`}>
                  <input type="checkbox" className="mr-2" checked={checked} onChange={() => {
                    const value = l.name; const next = checked ? selectedLayers.filter((x) => x !== value) : [...selectedLayers, value]; setSelectedLayers(next);
                  }} />
                  {l.title || l.name}
                </label>
              );
            })}
          </div>
        ) : (
          <div className="mt-2 flex flex-wrap gap-2">
            {collections.length === 0 && <div className="text-sm text-gray-500 dark:text-gray-400">— Lae /collections, et valida —</div>}
            {collections.map((c) => (
              <label key={c.id} className={`px-2 py-1 border rounded-full text-sm cursor-pointer ${selectedCollection === c.id ? "bg-blue-600 text-white dark:bg-blue-500" : ""}`}>
                <input type="radio" className="mr-2" checked={selectedCollection === c.id} onChange={() => setSelectedCollection(c.id)} />
                {c.title || c.id}
              </label>
            ))}
          </div>
        )}
      </div>

      {/* Parameters */}
      <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
        <h2 className="text-lg font-semibold">Parameetrid</h2>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mt-2">
          {/* CRS/SRS */}
          <div>
            <label className="text-sm font-medium">{isWMS && wmsVersion === "1.3.0" ? "CRS" : "SRS"} (lukus)</label>
            <input className="w-full border rounded-xl px-3 py-2" value={crs} readOnly />
          </div>
          {/* Width/Height */}
          <div className="grid grid-cols-2 gap-2">
            <div>
              <label className="text-sm font-medium">WIDTH</label>
              <input type="number" className="w-full border rounded-xl px-3 py-2" value={width} onChange={(e) => setWidth(Number(e.target.value))} />
            </div>
            <div>
              <label className="text-sm font-medium">HEIGHT</label>
              <input type="number" className="w-full border rounded-xl px-3 py-2" value={height} onChange={(e) => setHeight(Number(e.target.value))} />
            </div>
          </div>
          {/* Format/Transparent (WMS) */}
          {isWMS && (
            <>
              <div>
                <label className="text-sm font-medium">FORMAT</label>
                <input className="w-full border rounded-xl px-3 py-2" value={format} onChange={(e) => setFormat(e.target.value)} />
              </div>
              <div className="flex items-center gap-2">
                <label className="text-sm font-medium">TRANSPARENT</label>
                <input type="checkbox" checked={transparent} onChange={(e) => setTransparent(e.target.checked)} />
              </div>
              <div className="col-span-2">
                <label className="text-sm font-medium">STYLES (comma-separated)</label>
                <input className="w-full border rounded-xl px-3 py-2" value={styles} onChange={(e) => setStyles(e.target.value)} />
              </div>
            </>
          )}

          {/* WFS OUTPUTFORMAT */}
          {isWFS && (
            <div className="col-span-2">
              <label className="text-sm font-medium">WFS OUTPUTFORMAT</label>
              <select className="w-full border rounded-xl px-3 py-2" value={wfsOutputFormat} onChange={(e) => setWfsOutputFormat(e.target.value)}>
                <option value="">— vaikimisi —</option>
                {wfsFormats.map((f) => (<option key={f} value={f}>{f}</option>))}
              </select>
              <p className="text-xs text-gray-500 dark:text-gray-400 mt-1">Kui jätad tühjaks, ei lisata OUTPUTFORMAT parameetrit.</p>
            </div>
          )}

          {/* BBOX + presets */}
          <div className="col-span-2">
            <div className="flex items-center gap-2">
              <div className="flex-1">
                <label className="text-sm font-medium">BBOX (EPSG:3301, minx,miny,maxx,maxy)</label>
                <input className="w-full border rounded-xl px-3 py-2 font-mono" value={bbox} onChange={(e) => setBbox(e.target.value)} />
              </div>
              <div className="w-56">
                <label className="text-sm font-medium">BBOX preset</label>
                <select className="w-full border rounded-xl px-3 py-2" value={selectedBboxPreset} onChange={(e) => { const v = e.target.value; setSelectedBboxPreset(v); if (BBOX_PRESETS_3301[v]) setBbox(BBOX_PRESETS_3301[v]); }}>
                  {Object.keys(BBOX_PRESETS_3301).map((name) => (<option key={name} value={name}>{name}</option>))}
                </select>
              </div>
            </div>
            {/* Variant helper */}
            <BboxVariants bbox={bbox} onApply={(v) => setBbox(v)} />
          </div>

          {/* Custom params */}
          <div className="col-span-2">
            <label className="text-sm font-medium">Lisaparameetrid</label>
            {customParams.map((kv, idx) => (
              <div key={idx} className="grid grid-cols-2 gap-2 mt-1">
                <input className="border rounded-xl px-3 py-2" placeholder="key" value={kv.key} onChange={(e) => { const next = [...customParams]; next[idx] = { ...next[idx], key: e.target.value }; setCustomParams(next); }} />
                <input className="border rounded-xl px-3 py-2" placeholder="value" value={kv.value} onChange={(e) => { const next = [...customParams]; next[idx] = { ...next[idx], value: e.target.value }; setCustomParams(next); }} />
              </div>
            ))}
            <div className="mt-2 flex gap-2">
              <button className="px-3 py-2 rounded-xl border" onClick={() => setCustomParams([...customParams, { key: "", value: "" }])}>+ lisa rida</button>
              <button className="px-3 py-2 rounded-xl border" onClick={() => setCustomParams(customParams.filter((kv) => kv.key || kv.value))}>puhasta tühjad</button>
            </div>
          </div>
        </div>
      </div>

      {/* URL & actions */}
      <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
        <h2 className="text-lg font-semibold">URL eelvaade</h2>
        <textarea ref={urlAreaRef} className="w-full border rounded-xl px-3 py-2 font-mono h-28" value={urlPreview} readOnly />
        <div className="flex items-center gap-2 mt-2 flex-wrap">
          <label className="flex items-center gap-2 text-sm"><input type="checkbox" checked={autoUpdate} onChange={(e) => setAutoUpdate(e.target.checked)} /> Auto-uuendus</label>
          <button className="px-3 py-2 rounded-xl border" onClick={() => setUrlPreview(buildUrl())}>Uuenda URL</button>
          <button className="px-3 py-2 rounded-xl border" onClick={copyUrlToClipboard}>Kopeeri</button>
          <button className="px-3 py-2 rounded-xl border" onClick={selectUrlForManualCopy}>Vali URL</button>
          <a className="px-3 py-2 rounded-xl border" target="_blank" rel="noreferrer" href={urlPreview || "#"}>Ava uues sakis</a>
          <button className="ml-auto bg-blue-600 text-white rounded-xl px-3 py-2 hover:bg-blue-700 disabled:opacity-50" disabled={!urlPreview || loading} onClick={doRequest}>{loading ? "Laen…" : "Käivita päring"}</button>
        </div>
        {copyInfo && <p className="text-xs mt-2 text-gray-600 dark:text-gray-300">{copyInfo}</p>}
        {isWMS && request === "GetMap" && <p className="text-xs text-gray-500 dark:text-gray-400 mt-2">Pildi eelvaade allpool; vajadusel luba CORS või kasuta proxy't.</p>}
      </div>

      {/* Response viewer */}
      {(error || responseText || responseBlobUrl) && (
        <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
          <h2 className="text-lg font-semibold">Vastus</h2>
          {error && <pre className="text-red-600 whitespace-pre-wrap">{error}</pre>}
          {responseText && !error && <pre className="whitespace-pre-wrap text-xs overflow-auto max-h-[420px] border rounded-xl p-2">{responseText}</pre>}
          {responseBlobUrl && !error && <img alt="GetMap preview" className="mt-2 rounded-xl border" src={responseBlobUrl} />}
        </div>
      )}

      {/* Presets */}
      <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
        <h2 className="text-lg font-semibold">Presetid</h2>
        <div className="flex gap-2 mt-2">
          <input className="border rounded-xl px-3 py-2" placeholder="preseti nimi" value={newPresetName} onChange={(e) => setNewPresetName(e.target.value)} />
          <button className="px-3 py-2 rounded-xl border" onClick={savePreset}>Salvesta preset</button>
        </div>
        <div className="flex flex-wrap gap-2 mt-2">
          {Object.keys(presets).length === 0 && <div className="text-sm text-gray-500 dark:text-gray-400">— veel pole salvestatud —</div>}
          {Object.keys(presets).map((k) => (
            <span key={k} className="inline-flex items-center gap-2 px-2 py-1 rounded-full border text-sm">
              <button onClick={() => loadPreset(k)}>{k}</button>
              <button title="kustuta" onClick={() => { const next = { ...presets }; delete next[k]; setPresets(next); saveJson(PRESET_KEY, next); }}>✕</button>
            </span>
          ))}
        </div>
      </div>

      {/* Self-test panel */}
      <div className="border rounded-2xl p-4 border-neutral-300 dark:border-neutral-700">
        <h2 className="text-lg font-semibold">Self-test</h2>
        <p className="text-sm">{selfTest}</p>
      </div>
    </div>
  );
}