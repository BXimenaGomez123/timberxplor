<script>
    import mapboxgl from "mapbox-gl";
    import * as d3 from 'd3';
    import { onMount } from "svelte";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  
    mapboxgl.accessToken = "pk.eyJ1IjoiYnhpbWVuYWdvbWV6IiwiYSI6ImNtYXBodHd2dDBqNDQyanEzODlzcXZzdXUifQ.9K8alwSlgsUIZedt02AZ0w";
  
    let map;
  
    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-67.78, -10.02], 
            zoom: 6,
            style: "mapbox://styles/mapbox/streets-v12",
        });
        await new Promise(res => map.on("load", res));

        
        const data = await d3.json('/src/lib/data/caminos.json');
        const features = data.paths.map((path, idx) => ({
        type: 'Feature',
        geometry: {
            type: 'LineString',
            coordinates: path.nodes.map(n => [n.longitude, n.latitude])
        },
        properties: {
            weight: path.weight,
            id: idx
        }
        }));

        map.addSource('all-paths', {
            type: 'geojson',
            data: {
                type: 'FeatureCollection',
                features
            }
        });

        map.addLayer({
            id: 'paths-layer',
            type: 'line',
            source: 'all-paths',
            paint: {
                'line-color': '#000000',
                'line-width': 4,
                'line-opacity': 0.7
            }
        });

        data.paths.forEach(path =>
            path.nodes.forEach(node => {
            new mapboxgl.Marker({ color: {
                MANEJO: 'red',
                PTO_IBAMA: 'green',
                FINAL: 'blue'
                }[node.type] || 'white'
            })
                .setLngLat([node.longitude, node.latitude])
                .setPopup(new mapboxgl.Popup().setText(`${node.type}: ${node.id}`))
                .addTo(map);
            })
        );
    }
  
    onMount(() => {
      initMap();
    });
</script>
  

  
  <h1>Red de Rutas</h1>
  
  <div id="map" style="width:50%; height:600px;"></div>
  
 
  <style>
    @import url("$lib/global.css");
    #map { border: 1px solid #ccc; border-radius: 8px; }
</style>
