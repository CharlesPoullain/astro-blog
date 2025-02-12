---
title: 'Cloudflare Worker x Notion API'
description: 'Projet Cloudflare Worker x Notion API x d3'
pubDate: 'Feb 12 2025'
heroImage: '/blog-placeholder-3.jpg'
---

# Présentation du projet
## Stack technique
API Hono, Frontend Vue + d3

# Clouflare Worker

# Notion API

`Endpoint - Controller Hono`

```typescript
app.get("/api/notion/database/query", async (c) => {
  const { NOTION_API_KEY, NOTION_DATABASE_ID } = env<{
    NOTION_API_KEY: string;
    NOTION_DATABASE_ID: string;
  }>(c);

  const notionConnector = new NotionConnector(NOTION_API_KEY);
  const results = await notionConnector.queryDatabase(NOTION_DATABASE_ID);
  customLogger("Resultats:", `Nb: ${results.length}`);

  return c.json(results);
});
```

`Service Connector Notion`
```typescript
import { Client } from "@notionhq/client";

export default class NotionConnector {
  private notion: Client;

  constructor(apiKey: string) {
    this.notion = new Client({
      auth: apiKey,
    });
  }

  async queryDatabase(databaseId: string) {
    const response = await this.notion.databases.query({
      database_id: databaseId,
      sorts: [
        {
          property: "Name",
          direction: "ascending",
        },
      ],
    });
    return response.results;
  }
}
```

# Frontend - Rendu D3

`services/FetchPosts.js`
```javascript
export default class FetchPosts {
  static async execute() {
    const requestOptions = {
      method: 'GET'
    }
    const data = await fetch(import.meta.env.VITE_API_URL, requestOptions)
    return data.json()
  }
}
```


`components/GraphView.vue`
``` html

<script setup>
import { onMounted, ref } from 'vue'
import * as d3 from 'd3'
import FetchPosts from '../services/FetchPosts.js'

const graph = ref(null)
const nodes = ref([])
const links = ref([])

const fetchDatas = async () => {
  const data = await FetchPosts.execute()
  // import data from '../posts.json'
  nodes.value = data?.map((page) => ({
    id: page.id,
    title: page.properties.Name.title[0].plain_text
  }))

  data?.forEach((page) => {
    const sourceId = page.id
    const related = page.properties['Related base test'].relation
    related.forEach((relation) => {
      links.value.push({
        source: sourceId,
        target: relation.id
      })
    })
  })

  // Pour garantir des liens réciproques
  data?.forEach((page) => {
    const targetId = page.id
    const relatedBack = page.properties['Related back to base test'].relation
    relatedBack.forEach((relation) => {
      links.value.push({
        source: relation.id,
        target: targetId
      })
    })
  })
}

const createGraph = () => {
  const width = 800
  const height = 600
  // Specify the color scale.
  const color = d3.scaleOrdinal(d3.schemeCategory10)

  // The force simulation mutates links and nodes, so create a copy
  // so that re-evaluating this cell produces the same result.

  // Create a simulation with several forces.
  const simulation = d3
    .forceSimulation(nodes.value)
    .force(
      'link',
      d3
        .forceLink(links.value)
        .id((d) => d.id)
        .distance(100)
    )
    .force('charge', d3.forceManyBody().strength(-300))
    .force('center', d3.forceCenter(0, 0))
    .force('collision', d3.forceCollide().radius(50))
    .force('x', d3.forceX(0).strength(0.1))
    .force('y', d3.forceY(0).strength(0.1))

  // Create the SVG container.
  const svg = d3
    .select(graph.value)
    .append('svg')
    .attr('width', width)
    .attr('height', height)
    .attr('viewBox', [-width / 2, -height / 2, width, height])
    .attr('style', 'max-width: 100%; height: auto;')

  // Add a line for each link, and a circle for each node.
  const link = svg
    .append('g')
    .attr('stroke', '#999')
    .attr('stroke-opacity', 0.4)
    .selectAll('line')
    .data(links.value)
    .join('line')
    .attr('stroke-width', (d) => Math.sqrt(d.value))

  const node = svg
    .append('g')
    .attr('stroke', '#fff')
    .attr('stroke-width', 1.5)
    .selectAll('circle')
    .data(nodes.value)
    .join('circle')
    .attr('r', 10)
    .attr('fill', (d) => color(d.group))
    .attr('cursor', 'pointer')

  node.append('title').text((d) => d.id)

  const label = svg
    .append('g')
    .selectAll('text')
    .data(nodes.value)
    .enter()
    .append('text')
    .attr('dy', -3)
    .attr('dx', 12)
    .attr('cursor', 'default')
    .attr('fill', 'white')
    .text((d) => d.title)

  // Add a drag behavior.
  node.call(d3.drag().on('start', dragstarted).on('drag', dragged).on('end', dragended))

  // Set the position attributes of links and nodes each time the simulation ticks.
  simulation.on('tick', () => {
    link
      .attr('x1', (d) => d.source.x)
      .attr('y1', (d) => d.source.y)
      .attr('x2', (d) => d.target.x)
      .attr('y2', (d) => d.target.y)

    node.attr('cx', (d) => d.x).attr('cy', (d) => d.y)
    label.attr('x', (d) => d.x).attr('y', (d) => d.y)
  })

  // Reheat the simulation when drag starts, and fix the subject position.
  function dragstarted(event) {
    if (!event.active) simulation.alphaTarget(0.3).restart()
    event.subject.fx = event.subject.x
    event.subject.fy = event.subject.y
  }

  // Update the subject (dragged node) position during drag.
  function dragged(event) {
    event.subject.fx = event.x
    event.subject.fy = event.y
  }

  // Restore the target alpha so the simulation cools after dragging ends.
  // Unfix the subject position now that it’s no longer being dragged.
  function dragended(event) {
    if (!event.active) simulation.alphaTarget(0)
    event.subject.fx = null
    event.subject.fy = null
  }

  // When this cell is re-run, stop the previous simulation. (This doesn’t
  // really matter since the target alpha is zero and the simulation will
  // stop naturally, but it’s a good practice.)
  // invalidation.then(() => simulation.stop())
}

onMounted(async () => {
  await fetchDatas()
  createGraph()
})
</script>

<template>
  <div id="graph" ref="graph"></div>
</template>
<style scoped></style>

```

![rendu-cloudflare-notion](rendu-notion-d3.png)

# Résultat