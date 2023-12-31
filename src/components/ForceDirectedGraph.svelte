<script>
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import debounce from "lodash.debounce";

  let flavor = "";
  let expandedNodes = new Set();
  let nodes = [];
  let links = [];
  let simulation;
  let isDragging = false;
  let autoCompleteResults = [];
  let selectedIndex = -1;
  let originalSearch = "";
  let searchInput;

  onMount(() => {
    updateGraph();
    searchInput.focus();
  });

  async function expandNode(flavor) {
    const res = await fetch(
      `https://fdbackend-d0a756cc3435.herokuapp.com/recommendations?flavor=${flavor}`
    );
    const data = await res.json();

    // Check if recommendations are null
    if (data.recommendations === null) {
      return; // Exit the function early
    }

    if (!nodes.some((node) => node.name === data.flavor.toLowerCase())) {
      nodes.push({ name: data.flavor.toLowerCase(), nodeType: "Flavor" });
    }
    expandedNodes.add(data.flavor.toLowerCase());

    data.recommendations.forEach((rec) => {
      if (!nodes.some((node) => node.name === rec.name)) {
        nodes.push({ name: rec.name, nodeType: rec.nodeType });
      }
      if (
        !links.some(
          (link) =>
            link.source.name === data.flavor && link.target.name === rec.name
        )
      ) {
        links.push({
          source: data.flavor,
          target: rec.name,
          strength: rec.strength,
          relationshipType: rec.relationshipType,
        });
      }
    });
  }

  function collapseNode(flavor) {
    const normalizedFlavor = flavor.toLowerCase(); // Normalize the flavor
    expandedNodes.delete(normalizedFlavor);

    // Identify links that are connected to the flavor to be collapsed
    const linksToRemove = links.filter(
      (link) => link.source.name === flavor || link.target.name === flavor
    );

    // Identify nodes that are connected to the flavor to be collapsed
    const nodesToRemove = linksToRemove.map((link) =>
      link.source.name === flavor ? link.target.name : link.source.name
    );

    // Remove links connected to the flavor to be collapsed
    links = links.filter((link) => {
      return !(
        (link.source.name === flavor || link.target.name === flavor) &&
        !Array.from(expandedNodes).some(
          (expandedNode) =>
            link.source.name === expandedNode ||
            link.target.name === expandedNode
        )
      );
    });

    // Remove nodes that are not connected to any other expanded node and are not in the expandedNodes set
    nodes = nodes.filter((node) => {
      return (
        expandedNodes.has(node.name) ||
        node.name === flavor ||
        !nodesToRemove.includes(node.name) ||
        Array.from(expandedNodes).some((expandedNode) =>
          links.some(
            (link) =>
              (link.source.name === expandedNode &&
                link.target.name === node.name) ||
              (link.target.name === expandedNode &&
                link.source.name === node.name)
          )
        )
      );
    });
  }

  async function fetchDataAndUpdate(flavor) {
    const normalizedFlavor = flavor.toLowerCase();
    if (!expandedNodes.has(normalizedFlavor)) {
      await expandNode(normalizedFlavor);
    } else {
      collapseNode(normalizedFlavor);
    }
    updateGraph();
  }

  function updateGraph() {
    d3.select("#forceGraph").selectAll("*").remove();

    const width = window.innerWidth;
    const height = window.innerHeight;

    const colorMap = {
      Ingredient: "#fee07e",
      Taste: "#ecb5ca",
      Volume: "#f16767",
      Weight: "#ca90c0",
      Season: "#8cce91",
      Function: "#f79767",
      Technique: "#58c7e3",
      Related: "#d9c8ae",
    };

    const svg = d3
      .select("#forceGraph")
      .attr("preserveAspectRatio", "xMinYMin meet")
      .attr("viewBox", `0 0 ${window.innerWidth} ${window.innerHeight}`);

    const zoomGroup = svg.append("g"); // Define zoomGroup after svg

    const colorScale = d3.scaleLinear().domain([1, 4]).range(["#ccc", "#000"]); // Define colorScale

    if (!simulation) {
      // Initialize simulation if it's the first time
      simulation = d3
        .forceSimulation(nodes)
        .force(
          "link",
          d3
            .forceLink(links)
            .id((d) => d.name)
            .distance(100)
        )
        .force("charge", d3.forceManyBody().strength(-500))
        .force(
          "center",
          d3.forceCenter(window.innerWidth / 2, window.innerHeight / 2)
        );
    } else {
      // Update the existing simulation
      simulation.nodes(nodes); // Update nodes
      simulation.force("link").links(links); // Update links
    }

    // Restart the simulation
    simulation.alpha(1).restart();

    const link = zoomGroup
      .append("g")
      .selectAll("line")
      .data(links)
      .enter()
      .append("line")
      .attr("stroke", (d) => colorScale(d.strength))
      .attr("stroke-width", 1);

    const nodeGroup = zoomGroup
      .append("g")
      .selectAll("g.node")
      .data(nodes)
      .enter()
      .append("g")
      .attr("class", "node")
      .on("dblclick", (event, d) => {
        fetchDataAndUpdate(d.name);
      });

    // Use the color map to set the fill color based on node type
    nodeGroup
      .append("circle")
      .attr("r", 5)
      .attr("fill", (d) => colorMap[d.nodeType] || "#fee07e"); // Default to black if type is unknown

    nodeGroup
      .append("text")
      .text((d) => d.name)
      .attr("x", 6)
      .attr("y", 3)
      .attr("font-size", "12px")
      .attr("font-family", "Arial, Helvetica, sans-serif")
      .attr("pointer-events", "none");

    const zoom = d3
      .zoom()
      .scaleExtent([0.1, 10])
      .on("zoom", (event) => {
        if (!isDragging) {
          const currentZoomScale = event.transform.k;
          zoomGroup.attr("transform", event.transform);

          // Hide or show labels based on zoom level
          if (currentZoomScale < 0.7) {
            zoomGroup.selectAll("text").style("display", "none");
          } else {
            zoomGroup.selectAll("text").style("display", "block");
          }
        }
      });

    // Drag functions
    function dragstarted(event, d) {
      isDragging = true;
      if (!event.active) simulation.alphaTarget(0.3).restart();
      d.fx = d.x;
      d.fy = d.y;
    }

    function dragged(event, d) {
      d.fx = event.x;
      d.fy = event.y;
    }

    function dragended(event, d) {
      isDragging = false;
      if (!event.active) simulation.alphaTarget(0);
      d.fx = null;
      d.fy = null;
    }

    // Add drag behavior to nodes
    const drag = d3
      .drag()
      .on("start", dragstarted)
      .on("drag", dragged)
      .on("end", dragended);

    nodeGroup.call(drag);

    svg.call(zoom).on("dblclick.zoom", null);

    simulation.on("tick", () => {
      link
        .attr("x1", (d) => d.source.x)
        .attr("y1", (d) => d.source.y)
        .attr("x2", (d) => d.target.x)
        .attr("y2", (d) => d.target.y);

      nodeGroup.attr("transform", (d) => `translate(${d.x}, ${d.y})`);
    });
  }

  // Function to fetch autocomplete suggestions
  async function fetchAutoCompleteResults(flavor) {
    if (flavor.length < 2) {
      // Fetch suggestions if at least 2 characters are typed
      autoCompleteResults = [];
      return;
    }

    const res = await fetch(
      `https://fdbackend-d0a756cc3435.herokuapp.com/autocomplete?prefix=${flavor}`
    );
    autoCompleteResults = await res.json(); // Assuming the response is an array of strings
  }

  // Debounce the function to avoid too many requests
  const debounceFetchAutoCompleteResults = debounce(
    fetchAutoCompleteResults,
    200
  );

  // Watch for changes in the `flavor` variable and fetch autocomplete suggestions
  $: if (flavor) {
    debounceFetchAutoCompleteResults(flavor);
  }

  // Function to handle selecting an autocomplete result
  function selectAutoCompleteResult(result) {
    flavor = result;
    fetchDataAndUpdate(flavor); // You may want to fetch the data for the selected result
    autoCompleteResults = []; // Clear the autocomplete results
    flavor = ""; // Clear the search box after selecting a result
  }

  function handleKeyUp(event) {
    if (event.key === "Enter") {
      if (selectedIndex !== -1) {
        selectAutoCompleteResult(autoCompleteResults[selectedIndex]);
      } else {
        if (flavor.toLowerCase() === "clear") {
          clearGraph();
        } else {
          fetchDataAndUpdate(flavor.toLowerCase());
        }
      }
      originalSearch = "";
      flavor = "";
      autoCompleteResults = [];
      selectedIndex = -1;
    } else {
      originalSearch = flavor;
    }
  }

  function handleKeyDown(event) {
    if (
      event.key === "ArrowDown" &&
      selectedIndex < autoCompleteResults.length - 1
    ) {
      event.preventDefault();
      selectedIndex++;
    } else if (event.key === "ArrowUp") {
      event.preventDefault();
      if (selectedIndex > 0) {
        selectedIndex--;
      } else if (selectedIndex === 0) {
        selectedIndex = -1;
        flavor = originalSearch;
      }
    }
  }

  function clearGraph() {
    nodes.length = 0;
    links.length = 0;
    expandedNodes.clear();
    updateGraph();
    flavor = ""; // Clear the search box
  }

  window.addEventListener("resize", () => {
    updateGraph();
  });
</script>

<div id="search-container">
  <input
    type="text"
    bind:value={flavor}
    placeholder="Enter flavor"
    on:keyup={handleKeyUp}
    on:keydown={handleKeyDown}
    bind:this={searchInput}
  />

  <button on:click={clearGraph}>Clear</button>
  {#if autoCompleteResults.length}
    <ul>
      {#each autoCompleteResults as result, index}
        <li class:selected={index === selectedIndex}>
          <button
            class="autocomplete-button"
            on:click={() => selectAutoCompleteResult(result)}
            on:keyup={(event) =>
              event.key === "Enter" ? selectAutoCompleteResult(result) : null}
          >
            {result}
          </button>
        </li>
      {/each}
    </ul>
  {/if}
</div>
<svg id="forceGraph" />

<style>
  #search-container {
    position: absolute;
    top: 20px;
    left: 20px;
    z-index: 1; /* Make sure it appears above the SVG */
  }
  #forceGraph {
    background-color: #f6f7fb;
  }
  :global(body),
  :global(svg) {
    margin: 0;
    padding: 0;
  }

  /* Style for the autocomplete dropdown */
  .autocomplete-button {
    text-align: left; /* Align text to the left */
    border: none; /* Remove border */
    background: transparent; /* Make the background transparent */
    cursor: pointer; /* Make the cursor a hand when hovering over the button */
    width: 100%; /* Make the button take up the full width of the li */
    padding: 8px 12px; /* Apply padding to the button instead of the li */
  }

  ul {
    list-style-type: none;
    margin: 0;
    padding: 0;
    position: absolute;
    background-color: white;
    border: 1px solid #ddd;
    max-height: 200px;
    overflow-y: auto;
    overflow-x: hidden; /* Hide horizontal scrollbar */
    width: calc(100% - 2px); /* Adjust width to account for border */
  }

  li {
    cursor: pointer;
    width: 100%;
  }

  li:hover,
  .autocomplete-button:hover {
    background-color: #f5f5f5;
  }

  li.selected {
    background-color: #cccccc;
  }
</style>
