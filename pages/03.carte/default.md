---
title: carte
body_classes: 'title-center title-h1h2'
hide_page_title: false
show_sidebar: true
hide_git_sync_repo_link: false
process:
    markdown: true
    twig: true
menu: Carte
---

<div id="map" style="width: 100%; height: 600px;"></div>
Carte tirée de [meshmap.net](https://github.com/brianshea2/meshmap.net)

<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css" />
<link rel="stylesheet" href="https://unpkg.com/leaflet.markercluster/dist/MarkerCluster.css" />
<link rel="stylesheet" href="https://unpkg.com/leaflet.markercluster/dist/MarkerCluster.Default.css" />
<link rel="stylesheet" href="https://unpkg.com/leaflet-easybutton/src/easy-button.css" />
<link rel="stylesheet" href="https://unpkg.com/leaflet-search/dist/leaflet-search.src.css" />

<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script src="https://unpkg.com/leaflet.markercluster/dist/leaflet.markercluster.js"></script>
<script src="https://unpkg.com/leaflet-easybutton/src/easy-button.js"></script>
<script src="https://unpkg.com/leaflet-search/dist/leaflet-search.src.js"></script>
<script>
  let lastActiveNode
  let markersByNode = {}
  let nodesBySearchString = {}
  const ipinfoToken = 'aeb066758afd49'
  const updateInterval = 65000
  const precisionMargins = [
    11939464, 5969732, 2984866, 1492433, 746217, 373108, 186554, 93277,
    46639, 23319, 11660, 5830, 2915, 1457, 729, 364,
    182, 91, 46, 23, 11, 6, 3, 1,
    1, 0, 0, 0, 0, 0, 0, 0
  ]
  // encodes html reserved characters and ascii control characters
  const html = str => str
    ?.replace(/[\x00-\x1F]/g, c => `\\x${c.charCodeAt(0).toString(16).toUpperCase().padStart(2, '0')}`)
    .replace(/["&<>]/g, c => `&#${c.charCodeAt(0)};`)
  // makes more human-readable time duration strings
  const duration = d => {
    let s = ''
    if (d > 86400) {
      s += `${Math.floor(d / 86400)}d `
      d %= 86400
    }
    if (d > 3600) {
      s += `${Math.floor(d / 3600)}h `
      d %= 3600
    }
    s += `${Math.floor(d / 60)}min`
    return s
  }
  const since = t => `${duration(Date.now() / 1000 - t)} ago`
  // init map
  const map = L.map('map', {
    center: window.localStorage.getItem('center')?.split(',') ?? [25, 0],
    zoom: window.localStorage.getItem('zoom') ?? 2,
    attributionControl: false,
    zoomControl: false,
    worldCopyJump: true,
  })
  // add tiles
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: 'Map tiles from <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    maxZoom: 17,
  }).addTo(map)
  // add marker group
  const markers = L.markerClusterGroup({
    maxClusterRadius: zoom => zoom < 3 ? 40 : 30,
  }).addTo(map)
  // add node details layer (precision circle, relation lines)
  const detailsLayer = L.layerGroup().addTo(map)
  map.on('click', () => detailsLayer.clearLayers())
  // add search control
  map.addControl(new L.Control.Search({
    layer: markers,
    propertyName: 'searchString',
    initial: false,
    position: 'topleft',
    marker: false,
    moveToLocation: (_, s) => showNode(nodesBySearchString[s]),
  }))
  // add zoom control
  L.control.zoom({position: 'topright'}).addTo(map)
  // add geolocation control
  L.easyButton({
    position: 'topright',
    states: [
      {
        stateName: 'geolocation-button',
        title: 'Center map to current IP geolocation',
        icon: 'fa-crosshairs fa-lg',
        onClick: () => {
          fetch(`https://ipinfo.io/json?token=${ipinfoToken}`)
            .then(r => r.json())
            .then(({loc}) => loc && map.flyTo(loc.split(','), 10))
            .catch(e => console.error('Failed to set location:', e))
        },
      },
    ],
  }).addTo(map)
  // add attribution control
  const attribution = L.control.attribution({position: 'bottomright', prefix: false}).addTo(map)
  // updates attribution with given count of nodes
  const updateNodeCount = count => attribution.setPrefix(`
    ${count.toLocaleString()} nodes from
    <a href="https://meshtastic.org/docs/software/integrations/mqtt/#public-mqtt-server">Meshtastic MQTT</a>
  `)
  // track and store map position
  map.on('moveend', () => {
    const center = map.getCenter()
    window.localStorage.setItem('center', [center.lat, center.lng].join(','))
  })
  map.on('zoomend', () => {
    window.localStorage.setItem('zoom', map.getZoom())
  })
  // generates html for a node link
  const nodeLink = (num, label) => `<a href="#${num}" onclick="showNode(${num});return false;">${html(label)}</a>`
  // updates node map markers
  const updateNodes = data => {
    const popupWasOpen = lastActiveNode && markersByNode[lastActiveNode]?.isPopupOpen()
    const detailsLayerWasPopulated = detailsLayer.getLayers().length > 0
    markersByNode = {}
    nodesBySearchString = {}
    markers.clearLayers()
    detailsLayer.clearLayers()
    let reactivate = () => {}
    const relationsByNode = {}
    Object.entries(data).forEach(([nodeNum, node]) => {
      const {
        longName, shortName, hwModel, role,
        fwVersion, region, modemPreset, hasDefaultCh, onlineLocalNodes,
        latitude, longitude, altitude, precision,
        batteryLevel, voltage, chUtil, airUtilTx, uptime,
        temperature, relativeHumidity, barometricPressure, lux,
        windDirection, windSpeed, windGust, radiation, rainfall1, rainfall24,
        neighbors, seenBy
      } = node
      const id = `!${Number(nodeNum).toString(16)}`
      const position = L.latLng([latitude, longitude].map(x => x / 10000000))
      const seenByNums = new Set(
        Object.keys(seenBy)
          .map(topic => topic.match(/\/!([0-9a-f]+)$/))
          .filter(match => match)
          .map(match => parseInt(match[1], 16))
      )
      relationsByNode[nodeNum] ??= {}
      seenByNums.forEach(seenByNum => {
        relationsByNode[seenByNum] ??= {}
        const relationObj = relationsByNode[nodeNum][seenByNum] ?? relationsByNode[seenByNum][nodeNum] ?? {}
        relationsByNode[nodeNum][seenByNum] = relationObj
        relationsByNode[seenByNum][nodeNum] = relationObj
        relationObj.mqtt = true
      })
      if (neighbors) {
        Object.keys(neighbors).forEach(neighborNum => {
          relationsByNode[neighborNum] ??= {}
          const relationObj = relationsByNode[nodeNum][neighborNum] ?? relationsByNode[neighborNum][nodeNum] ?? {}
          relationsByNode[nodeNum][neighborNum] = relationObj
          relationsByNode[neighborNum][nodeNum] = relationObj
          relationObj.neighbor = true
        })
      }
      const drawNodeDetails = () => {
        detailsLayer.clearLayers()
        // precision circle
        if (precision && precisionMargins[precision-1]) {
          L.circle(position, {radius: precisionMargins[precision-1], color: '#ffa932'}).addTo(detailsLayer)
        }
        // relation lines
        Object.entries(relationsByNode[nodeNum]).forEach(([relatedNum, relationObj]) => {
          if (data[relatedNum] === undefined) {
            return
          }
          const relatedPosition = L.latLng([data[relatedNum].latitude, data[relatedNum].longitude].map(x => x / 10000000))
          const relationContent = `
            <table><tbody>
            <tr><th>Nodes</th><td>${nodeLink(nodeNum, id)}, ${nodeLink(relatedNum, `!${Number(relatedNum).toString(16)}`)}</td></tr>
            <tr><th>Relation type</th><td>${[relationObj.neighbor && 'Neighbor', relationObj.mqtt && 'MQTT uplink'].filter(v => v).join(', ')}</td></tr>
            <tr><th>Distance</th><td>${Math.round(map.distance(position, relatedPosition)).toLocaleString()} m</td></tr>
            ${neighbors?.[relatedNum]?.snr ? `<tr><th>SNR</th><td>${neighbors[relatedNum].snr} dB</td></tr>` : ''}
            </tbody></table>
          `
          L.polyline([position, relatedPosition])
            .bindTooltip(relationContent, {opacity: 0.95, sticky: true})
            .on('click', () => showNode(relatedNum))
            .addTo(detailsLayer)
        })
      }
      const lastSeen = Math.max(...Object.values(seenBy))
      const opacity = 1.0 - (Date.now() / 1000 - lastSeen) / 129600
      const tooltipContent = `${html(longName)} (${html(shortName)}) ${since(lastSeen)}`
      const popupContent = `
        <div class="title">${html(longName)} (${html(shortName)})</div>
        <div>${nodeLink(nodeNum, id)} | ${html(role)} | ${html(hwModel)}</div>
        <table><tbody>
        ${uptime             ? `<tr><th>Uptime</th><td>${duration(uptime)}</td></tr>`                                : ''}
        ${batteryLevel       ? `<tr><th>Power</th><td>${batteryLevel > 100 ? 'Plugged in' : `${batteryLevel}%`}` +
                               `${voltage ? ` (${voltage.toFixed(2)}V)` : ''}</td></tr>`                             : ''}
        ${fwVersion          ? `<tr><th>Firmware</th><td>${html(fwVersion)}</td></tr>`                               : ''}
        ${region             ? `<tr><th>LoRa config</th><td>${html(region)} / ${html(modemPreset)}</td></tr>`        : ''}
        ${chUtil             ? `<tr><th>ChUtil</th><td>${chUtil.toFixed(2)}%</td></tr>`                              : ''}
        ${airUtilTx          ? `<tr><th>AirUtilTX</th><td>${airUtilTx.toFixed(2)}%</td></tr>`                        : ''}
        ${onlineLocalNodes   ? `<tr><th>Local nodes</th><td>${onlineLocalNodes}</td></tr>`                           : ''}
        ${precision && precisionMargins[precision-1] ? `<tr><th>Map privacy</th><td>` +
                               `&#177;${precisionMargins[precision-1].toLocaleString()} m (orange circle)</td></tr>` : ''}
        ${altitude           ? `<tr><th>Altitude</th><td>${altitude.toLocaleString()} m above MSL</td></tr>`         : ''}
        ${temperature        ? `<tr><th>Temperature</th><td>${temperature.toFixed(1)}&#8451; / ` +
                               `${(temperature * 1.8 + 32).toFixed(1)}&#8457;</td></tr>`                             : ''}
        ${relativeHumidity   ? `<tr><th>Relative humidity</th><td>${Math.round(relativeHumidity)}%</td></tr>`        : ''}
        ${barometricPressure ? `<tr><th>Barometric pressure</th><td>${Math.round(barometricPressure)} hPa</td></tr>` : ''}
        ${windDirection || windSpeed ? `<tr><th>Wind</th><td>` +
                                (windDirection ? `${windDirection}&#176;` : '') +
                                (windDirection && windSpeed ? ' @ ' : '') +
                                (windSpeed ? `${(windSpeed * 3.6).toFixed(1)}` : '') +
                                (windSpeed && windGust ? ` G ${(windGust * 3.6).toFixed(1)}` : '') +
                                (windSpeed ? ' km/h' : '') +
                                `</td></tr>`                                                                         : ''}
        ${lux                ? `<tr><th>Lux</th><td>${Math.round(lux)} lx</td></tr>`                                 : ''}
        ${radiation          ? `<tr><th>Radiation</th><td>${radiation.toFixed(2)} µR/h</td></tr>`                    : ''}
        ${rainfall1 || rainfall24 ? `<tr><th>Rainfall</th><td>` +
                                (rainfall1 ? `${rainfall1.toFixed(2)} mm/h` : '') +
                                (rainfall1 && rainfall24 ? ', ' : '') +
                                (rainfall24 ? `${rainfall24.toFixed(2)} mm/24h` : '') +
                                `</td></tr>`                                                                         : ''}
        </tbody></table>
        <table><thead>
        <tr><th>Last seen</th><th>via</th><th>MQTT root</th><th>channel</th></tr>
        </thead><tbody>
        ${Array.from(
          new Map(
            Object.entries(seenBy)
              .map(([topic, seen]) => (match => ({seen, via: match[3] ?? id, root: match[1], chan: match[2]}))(
                topic.match(/^(.*)(?:\/2\/e\/(.*)\/(![0-9a-f]+)|\/2\/map\/)$/s)
              ))
              .sort((a, b) => a.seen - b.seen)
              .map(v => [v.via, v])
          ).values(),
          ({seen, via, root, chan}) => `
            <tr>
            <td>${since(seen)}</td>
            <td>${chan ? ((n, l) => data[n] ? nodeLink(n, l) : l)(parseInt(via.slice(1), 16), via === id ? 'self' : via) : 'MapReport'}</td>
            <td class="break">${html(root)}</td>
            <td class="break">${html(chan ?? 'n/a')}</td>
            </tr>
          `
        ).reverse().join('')}
        </tbody></table>
      `
      const searchString = `${longName} (${shortName}) ${id}`
      nodesBySearchString[searchString] = nodeNum
      markersByNode[nodeNum] = L.marker(position, {alt: 'Node', opacity, searchString})
        .bindTooltip(tooltipContent, {opacity: 0.95})
        .bindPopup(popupContent, {maxWidth: 600})
        .on('popupopen', () => {
          // set last active
          lastActiveNode = nodeNum
          // set URL fragment
          history.replaceState(null, '', `#${nodeNum}`)
          // draw precision circle, relation lines
          drawNodeDetails()
        })
      if (nodeNum === lastActiveNode) {
        if (popupWasOpen) {
          reactivate = () => {
            const cluster = markers.getVisibleParent(markersByNode[nodeNum])
            if (typeof cluster?.spiderfy === 'function') {
              cluster.spiderfy()
            }
            markersByNode[nodeNum].openPopup()
          }
        } else if (detailsLayerWasPopulated) {
          reactivate = drawNodeDetails
        }
      }
    })
    markers.addLayers(Object.values(markersByNode))
    reactivate()
  }
  // fetches node data, updates map, repeats
  const drawMap = async () => {
    try {
      await fetch('https://node-red.admin.agrimetrie.ca/nodes.json').then(r => r.json()).then(updateNodes)
      updateNodeCount(Object.keys(markersByNode).length)
    } catch (e) {
      console.error('Failed to update nodes:', e)
    }
    setTimeout(() => {
      if (document.hidden) {
        document.addEventListener('visibilitychange', drawMap, {once: true})
      } else {
        drawMap()
      }
    }, updateInterval)
  }
  // pans and zooms map to node and opens popup
  const showNode = nodeNum => {
    if (markersByNode[nodeNum] === undefined) {
      return false
    }
    map.panTo(markersByNode[nodeNum].getLatLng())
    setTimeout(() => {
      markers.zoomToShowLayer(markersByNode[nodeNum], () => {
        markersByNode[nodeNum].openPopup()
      })
    }, 300)
    return true
  }
  // keep URL fragment in sync
  window.addEventListener('hashchange', () => {
    if (window.location.hash && !showNode(window.location.hash.slice(1))) {
      history.replaceState(null, '', window.location.pathname)
    }
    if (!window.location.hash) {
      map.closePopup()
      detailsLayer.clearLayers()
    }
  })
  map.on('popupclose', () => {
    if (window.location.hash) {
      history.replaceState(null, '', window.location.pathname)
    }
  })
  // let's go!!!
  drawMap().then(() => {
    if (window.location.hash && !showNode(window.location.hash.slice(1))) {
      history.replaceState(null, '', window.location.pathname)
    }
  })
</script>