// SkyTrack Radar: servidor de posiciones (función de Vercel, Node.js, sin dependencias).
// GET /api/aeronaves?lat=18.9&lon=-70.2&radio=250
//
// El navegador nunca habla con los proveedores: esta función consulta, valida, normaliza,
// guarda en caché unos segundos y limita solicitudes. Ningún proveedor de esta fase usa clave;
// cuando alguno la necesite se leerá de process.env y nunca saldrá en una respuesta.

const VERSION = '0.4.0';
const AGENTE = `SkyTrackRadar/${VERSION} (uso personal, no comercial)`;
const SEG_CACHE = 5;            // una misma zona no se consulta al proveedor más de una vez cada 5 s
const MS_ESPERA = 7000;         // tiempo máximo por proveedor
const MS_ENTRE_LLAMADAS = 1100; // los proveedores ADS-B comunitarios piden 1 solicitud por segundo
const MS_CASTIGO = 60000;       // pausa de un proveedor tras responder 429
const MAX_POR_IP = 40;          // solicitudes por minuto y por IP (mejor esfuerzo: la memoria es por instancia)
const RADIO_MAX = 250;
const MAX_AERONAVES = 1500;

// Designadores OACI de aerolíneas frecuentes. Solo sirven para rotular; la ficha avisa de que es una deducción.
const AEROLINEAS = {
  AAL: 'American Airlines', JBU: 'JetBlue', DAL: 'Delta Air Lines', UAL: 'United Airlines', NKS: 'Spirit Airlines',
  FFT: 'Frontier Airlines', SWA: 'Southwest Airlines', DWI: 'Arajet', CMP: 'Copa Airlines', AVA: 'Avianca',
  IBE: 'Iberia', AEA: 'Air Europa', AFR: 'Air France', BAW: 'British Airways', DLH: 'Lufthansa', KLM: 'KLM',
  TAP: 'TAP Air Portugal', ACA: 'Air Canada', ROU: 'Air Canada Rouge', WJA: 'WestJet', TSC: 'Air Transat',
  SWG: 'Sunwing Airlines', CFG: 'Condor', EDW: 'Edelweiss Air', TOM: 'TUI Airways', UPS: 'UPS Airlines', FDX: 'FedEx Express',
};
const TURBOHELICES_REGIONALES = new Set(['AT43', 'AT45', 'AT72', 'AT75', 'AT76', 'DH8A', 'DH8B', 'DH8C', 'DH8D', 'SF34', 'E120', 'JS31', 'JS32', 'JS41', 'B190', 'L410']);

const num = (v) => (typeof v === 'number' && Number.isFinite(v) ? v : null);
const texto = (v, max = 40) => (typeof v === 'string' && v.trim() ? v.trim().slice(0, max) : null);

// La fuente no dice si un vuelo es comercial o privado: se estima y se marca como estimación.
function clasificar(indicativo, tipo, militar) {
  if (militar) return 'militar';
  const m = indicativo && /^([A-Z]{3})\d/.exec(indicativo);
  if (m) return tipo && TURBOHELICES_REGIONALES.has(tipo) ? 'regional' : 'comercial';
  return 'privado';
}

// Formato «ADS-B Exchange v2» que comparten Airplanes.live y adsb.fi.
function normalizarReadsb(cuerpo, fuente, ahora) {
  const lista = cuerpo && Array.isArray(cuerpo.ac) ? cuerpo.ac : cuerpo && Array.isArray(cuerpo.aircraft) ? cuerpo.aircraft : null;
  if (!lista) throw new Error('Respuesta sin lista de aeronaves');
  const vuelos = [];
  for (const a of lista.slice(0, MAX_AERONAVES)) {
    const hex = texto(a && a.hex, 8);
    if (!hex || hex.startsWith('~')) continue; // sin identificador fiable
    const lat = num(a.lat), lon = num(a.lon);
    const conPos = lat !== null && lon !== null && Math.abs(lat) <= 90 && Math.abs(lon) <= 180;
    const enTierra = a.alt_baro === 'ground';
    const indicativo = texto(a.flight, 10);
    const tipo = texto(a.t, 6);
    const prefijo = indicativo && /^([A-Z]{3})\d/.exec(indicativo);
    const aerolinea = prefijo && AEROLINEAS[prefijo[1]] ? AEROLINEAS[prefijo[1]] : null;
    const edadPos = num(a.seen_pos) ?? num(a.seen);
    vuelos.push({
      id: hex.toLowerCase(), hex: hex.toUpperCase(),
      vuelo: indicativo, aerolinea, aerolineaDeducida: !!aerolinea,
      matricula: texto(a.r, 12), tipo, modelo: texto(a.desc, 60),
      origen: null, destino: null, salida: null, llegada: null, // las redes ADS-B no informan la ruta
      estado: enTierra ? 'en-tierra' : conPos ? 'en-vuelo' : 'desconocido',
      cat: clasificar(indicativo, tipo, (num(a.dbFlags) & 1) === 1), catEstimada: true,
      lat: conPos ? lat : null, lon: conPos ? lon : null,
      alt: enTierra ? 0 : num(a.alt_baro) ?? num(a.alt_geom),
      gs: num(a.gs), vs: num(a.baro_rate) ?? num(a.geom_rate), rumbo: num(a.track) ?? num(a.true_heading),
      transpondedor: texto(a.squawk, 4),
      recibido: conPos && edadPos !== null ? Math.round(ahora - edadPos * 1000) : null,
      fuente, simulado: false,
    });
  }
  return vuelos;
}

// OpenSky entrega cada aeronave como una lista de valores en unidades métricas.
function normalizarOpenSky(cuerpo, fuente, ahora) {
  if (!cuerpo || !('states' in cuerpo)) throw new Error('Respuesta sin estados');
  const vuelos = [];
  for (const s of (cuerpo.states || []).slice(0, MAX_AERONAVES)) {
    if (!Array.isArray(s)) continue;
    const hex = texto(s[0], 8);
    if (!hex) continue;
    const lon = num(s[5]), lat = num(s[6]);
    const conPos = lat !== null && lon !== null;
    const enTierra = s[8] === true;
    const indicativo = texto(s[1], 10);
    const prefijo = indicativo && /^([A-Z]{3})\d/.exec(indicativo);
    const aerolinea = prefijo && AEROLINEAS[prefijo[1]] ? AEROLINEAS[prefijo[1]] : null;
    const m = num(s[7]) ?? num(s[13]);
    vuelos.push({
      id: hex.toLowerCase(), hex: hex.toUpperCase(),
      vuelo: indicativo, aerolinea, aerolineaDeducida: !!aerolinea,
      matricula: null, tipo: null, modelo: null,
      origen: null, destino: null, salida: null, llegada: null,
      estado: enTierra ? 'en-tierra' : conPos ? 'en-vuelo' : 'desconocido',
      cat: clasificar(indicativo, null, false), catEstimada: true,
      lat: conPos ? lat : null, lon: conPos ? lon : null,
      alt: enTierra ? 0 : m === null ? null : Math.round(m / 0.3048),
      gs: num(s[9]) === null ? null : Math.round(s[9] * 1.943844),
      vs: num(s[11]) === null ? null : Math.round((s[11] * 196.8504) / 64) * 64,
      rumbo: num(s[10]), transpondedor: texto(s[14], 4),
      recibido: conPos && num(s[3]) !== null ? s[3] * 1000 : null,
      fuente, simulado: false,
    });
  }
  return vuelos;
}

// Caja geográfica que contiene un círculo de radio dado (para OpenSky, que consulta por caja).
function cajaDe(lat, lon, nm) {
  const dLat = nm / 60;
  const dLon = nm / (60 * Math.max(0.2, Math.cos((lat * Math.PI) / 180)));
  return { sur: Math.max(-90, lat - dLat), norte: Math.min(90, lat + dLat), oeste: Math.max(-180, lon - dLon), este: Math.min(180, lon + dLon) };
}

// Orden de preferencia. Cambiar de proveedor es editar esta lista.
const PROVEEDORES = [
  { id: 'airplanes-live', nombre: 'Airplanes.live',
    url: (q) => `https://api.airplanes.live/v2/point/${q.lat}/${q.lon}/${q.radio}`, normalizar: normalizarReadsb },
  { id: 'adsb-fi', nombre: 'adsb.fi',
    url: (q) => `https://opendata.adsb.fi/api/v2/lat/${q.lat}/lon/${q.lon}/dist/${q.radio}`, normalizar: normalizarReadsb },
  { id: 'opensky', nombre: 'OpenSky Network',
    url: (q) => { const c = cajaDe(q.lat, q.lon, q.radio); return `https://opensky-network.org/api/states/all?lamin=${c.sur.toFixed(3)}&lomin=${c.oeste.toFixed(3)}&lamax=${c.norte.toFixed(3)}&lomax=${c.este.toFixed(3)}`; },
    normalizar: normalizarOpenSky },
];

// Memoria de la instancia. En un entorno sin servidor puede haber varias instancias, así que
// estos límites son una protección razonable, no una garantía absoluta.
const memoria = { cache: new Map(), ultima: new Map(), castigo: new Map(), ips: new Map() };

function leerParametros(url) {
  const p = new URL(url, 'http://local').searchParams;
  const pide = (clave, defecto) => (p.has(clave) ? Number(p.get(clave)) : defecto);
  const lat = pide('lat', 18.9), lon = pide('lon', -70.2), radio = pide('radio', RADIO_MAX);
  if (!Number.isFinite(lat) || lat < -90 || lat > 90) return { error: 'lat debe estar entre -90 y 90' };
  if (!Number.isFinite(lon) || lon < -180 || lon > 180) return { error: 'lon debe estar entre -180 y 180' };
  if (!Number.isFinite(radio) || radio < 1 || radio > RADIO_MAX) return { error: `radio debe estar entre 1 y ${RADIO_MAX} millas náuticas` };
  // Se redondea para que mapas casi iguales compartan la misma entrada de caché.
  return { lat: Math.round(lat * 4) / 4, lon: Math.round(lon * 4) / 4, radio: Math.min(RADIO_MAX, Math.ceil(radio / 25) * 25) };
}

function ipExcedida(ip, ahora) {
  const minuto = Math.floor(ahora / 60000);
  const r = memoria.ips.get(ip);
  if (!r || r.minuto !== minuto) { memoria.ips.set(ip, { minuto, n: 1 }); if (memoria.ips.size > 5000) memoria.ips.clear(); return false; }
  r.n += 1;
  return r.n > MAX_POR_IP;
}

async function consultar(prov, q, ahora, traer) {
  if ((memoria.castigo.get(prov.id) || 0) > ahora) return { id: prov.id, nombre: prov.nombre, ok: false, motivo: 'cuota' };
  if (ahora - (memoria.ultima.get(prov.id) || 0) < MS_ENTRE_LLAMADAS) return { id: prov.id, nombre: prov.nombre, ok: false, motivo: 'ritmo' };
  memoria.ultima.set(prov.id, ahora);
  const control = new AbortController();
  const reloj = setTimeout(() => control.abort(), MS_ESPERA);
  try {
    const r = await traer(prov.url(q), { headers: { 'User-Agent': AGENTE, Accept: 'application/json' }, signal: control.signal });
    if (r.status === 429) { memoria.castigo.set(prov.id, ahora + MS_CASTIGO); return { id: prov.id, nombre: prov.nombre, ok: false, motivo: 'cuota' }; }
    if (!r.ok) return { id: prov.id, nombre: prov.nombre, ok: false, motivo: 'http-' + r.status };
    const vuelos = prov.normalizar(await r.json(), prov.nombre, ahora);
    return { id: prov.id, nombre: prov.nombre, ok: true, vuelos };
  } catch (e) {
    // Nunca se devuelve el mensaje interno: podría contener direcciones o cabeceras.
    console.error('[aeronaves]', prov.id, e && e.name, e && e.message);
    return { id: prov.id, nombre: prov.nombre, ok: false, motivo: e && e.name === 'AbortError' ? 'sin-respuesta' : 'respuesta-invalida' };
  } finally { clearTimeout(reloj); }
}

function json(estado, cuerpo, extra = {}) {
  return new Response(JSON.stringify(cuerpo), {
    status: estado,
    headers: { 'Content-Type': 'application/json; charset=utf-8', 'X-Content-Type-Options': 'nosniff', ...extra },
  });
}

async function atender(request, { traer = fetch, ahora = Date.now() } = {}) {
  const q = leerParametros(request.url);
  if (q.error) return json(400, { error: 'parametros', detalle: q.error }, { 'Cache-Control': 'no-store' });

  const ip = (request.headers.get('x-forwarded-for') || 'desconocida').split(',')[0].trim();
  if (ipExcedida(ip, ahora)) return json(429, { error: 'cuota', detalle: 'Demasiadas solicitudes desde este dispositivo. Espera un minuto.' }, { 'Retry-After': '60', 'Cache-Control': 'no-store' });

  const clave = `${q.lat},${q.lon},${q.radio}`;
  const guardado = memoria.cache.get(clave);
  if (guardado && ahora - guardado.recibido < SEG_CACHE * 1000) {
    return json(200, { ...guardado, enCache: true }, { 'Cache-Control': `public, s-maxage=${SEG_CACHE}, stale-while-revalidate=10` });
  }

  const intentos = [];
  for (const prov of PROVEEDORES) {
    const r = await consultar(prov, q, ahora, traer);
    intentos.push({ id: r.id, nombre: r.nombre, ok: r.ok, motivo: r.motivo || null });
    if (r.ok) {
      const cuerpo = { vuelos: r.vuelos, fuente: r.nombre, simulado: false, recibido: ahora, consulta: q, proveedores: intentos, version: VERSION };
      memoria.cache.set(clave, cuerpo);
      if (memoria.cache.size > 200) memoria.cache.delete(memoria.cache.keys().next().value);
      return json(200, cuerpo, { 'Cache-Control': `public, s-maxage=${SEG_CACHE}, stale-while-revalidate=10` });
    }
  }
  // Ningún proveedor respondió. Si queda una copia reciente (menos de 2 min) se entrega marcada como antigua.
  if (guardado && ahora - guardado.recibido < 120000) return json(200, { ...guardado, enCache: true, proveedores: intentos }, { 'Cache-Control': 'no-store' });
  const todosCuota = intentos.every((i) => i.motivo === 'cuota' || i.motivo === 'ritmo');
  return json(todosCuota ? 429 : 502, { error: todosCuota ? 'cuota' : 'proveedores', detalle: 'Ninguna fuente de posiciones respondió.', proveedores: intentos },
    { 'Cache-Control': 'no-store', ...(todosCuota ? { 'Retry-After': '30' } : {}) });
}

export function GET(request) { return atender(request); }

// Solo para las pruebas automáticas.
export const _pruebas = { atender, normalizarReadsb, normalizarOpenSky, leerParametros, cajaDe, clasificar, memoria, PROVEEDORES };
