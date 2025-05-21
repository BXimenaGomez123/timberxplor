<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import mapboxgl from 'mapbox-gl';
    import 'mapbox-gl/dist/mapbox-gl.css';
  
    mapboxgl.accessToken = 'pk.eyJ1IjoiYnhpbWVuYWdvbWV6IiwiYSI6ImNtYXBodHd2dDBqNDQyanEzODlzcXZzdXUifQ.9K8alwSlgsUIZedt02AZ0w';
  
    let width = 0, height = 0;
    let svg, map, data;
  
    function updateSize() {
      width = window.innerWidth;
      height = window.innerHeight - 80;
      if (svg) svg.attr('width', width / 2).attr('height', height);
      if (map) map.resize();
    }
    


    function buildGraph() {
      // 1) Preparar nodos y enlaces
      const nodesMap = new Map();
      const links = [];
  
      data.paths.forEach((path, pi) => {
        path.nodes.forEach((n, i) => {
          if (!nodesMap.has(n.id)) {
            nodesMap.set(n.id, { ...n, pathIndex: pi });
          }
          if (i + 1 < path.nodes.length) {
            links.push({
              source: path.nodes[i].id,
              target: path.nodes[i + 1].id
            });
          }
        });
      });
  
      const nodes = Array.from(nodesMap.values());
  
      // 2) Prepara el SVG
      svg = d3.select('svg').attr('width', width / 2).attr('height', height);
      svg.selectAll('*').remove();
      svg.append('defs').append('marker')
        .attr('id', 'arrow').attr('viewBox', '0 -5 10 10')
        .attr('refX', 15).attr('refY', 0)
        .attr('markerWidth', 6).attr('markerHeight', 6)
        .attr('orient', 'auto')
        .append('path').attr('d', 'M0,-5L10,0L0,5').attr('fill', '#999');
  
      // 3) Simulación de fuerzas
      const sim = d3.forceSimulation(nodes)
        .force('link', d3.forceLink(links).id(d => d.id).distance(100))
        .force('charge', d3.forceManyBody().strength(-300))
        .force('center', d3.forceCenter((width / 2) / 2, height / 2));
  
      // 4) Dibuja enlaces
      const link = svg.append('g').attr('stroke', '#999')
        .selectAll('line').data(links).join('line')
        .attr('stroke-width', 2)
        .attr('marker-end', 'url(#arrow)');
  
      // 5) Dibuja nodos y click handler
      const node = svg.append('g')
        .selectAll('g').data(nodes).join('g')
        .call(d3.drag()
          .on('start', e => { if (!e.active) sim.alphaTarget(0.3).restart(); e.subject.fx = e.subject.x; e.subject.fy = e.subject.y; })
          .on('drag', e => { e.subject.fx = e.x; e.subject.fy = e.y; })
          .on('end', e => { if (!e.active) sim.alphaTarget(0); e.subject.fx = null; e.subject.fy = null; })
        )
        .on('click', (_, d) => {
          if (d.type === 'MANEJO') {
            renderMapForPath(d.pathIndex);
          }
        });
  
      node.append('circle')
        .attr('r', 8)
        .attr('fill', d => d.type === 'MANEJO' ? '#2ca02c' : '#ccc');
  
      node.append('text')
        .text(d => d.id)
        .attr('x', 10)
        .attr('y', 3)
        .attr('fill', '#fff')
        .style('font-size', '10px');
  
      // 6) Tick de la simulación
      sim.on('tick', () => {
        link
          .attr('x1', d => d.source.x).attr('y1', d => d.source.y)
          .attr('x2', d => d.target.x).attr('y2', d => d.target.y);
        node.attr('transform', d => `translate(${d.x},${d.y})`);
      });
    }
  
  
    function renderMapForPath(pathIndex) {
      const path = data.paths[pathIndex];
      const coords = path.nodes.map(n => [n.longitude, n.latitude]);
  
      // 1) actualiza la fuente 'selected-path'
      map.getSource('selected-path').setData({
        type: 'FeatureCollection',
        features: [{
          type: 'Feature',
          geometry: { type: 'LineString', coordinates: coords }
        }]
      });
  
      // 2) centra el mapa
      map.fitBounds(coords, { padding: 40 });
    }
  
  
    onMount(async () => {
      
        map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v12',
        center: [-67.78, -10.02],
        zoom: 6
        });
        await new Promise(r => map.on('load', r));

        
        data = await d3.json(`${import.meta.env.BASE_URL}data/caminos.json`);

        
        map.addSource('selected-path', {
        type: 'geojson',
        data: { type: 'FeatureCollection', features: [] }
        });
        map.addLayer({
        id: 'selected-path-layer',
        type: 'line',
        source: 'selected-path',
        paint: {
            'line-color': '#FFD700',
            'line-width': 5
        }
        });

        
        window.addEventListener('resize', updateSize);
        updateSize();

        const colorPorTipo = {
            MANEJO: 'red',
            PTO_IBAMA: 'green',
            FINAL: 'blue'
        };

        buildGraph();
    });
  
  </script>
  
  <style>
    .wrapper { display: flex; }
    svg { background: #1c1c1c; }
    #map { flex: 1; height: 100vh; border: 1px solid #ccc; border-radius: 8px; }
  </style>
  
  <div class="wrapper">
    <svg></svg>
    <div id="map"></div>
  </div>
  