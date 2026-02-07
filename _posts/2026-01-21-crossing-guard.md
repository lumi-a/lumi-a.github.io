---
layout: default
title: Crossing Guard
---

<style>
  svg {
    max-width: 550px;
    width: 100%;
    height: auto;
    display: block;
    margin: 2em auto;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  svg.small {
    max-width: 425px;
  }
  svg.tiny {
    max-width: 350px;
  }
  svg circle.point.red {
    fill: #E67; /* // https://sronpersonalpages.nl/~pault/ */
  }
  svg circle.point.blue {
    fill: #47A; 
  }
  svg circle.point.active {
    stroke: #888;
    stroke-width: 0.005;
  }
  svg line.connection {
    stroke: #888;
    stroke-width: 0.005;
    stroke-dasharray: 1.5 1.5;
    transition: stroke-dashoffset .5s, stroke .1s;
  }
  svg line.connection.invalid {
    stroke: #A37;
  }
  div.center {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5em;
  }
</style>
<script>
  const svg_states = new Map()

  function create_interactive_svg(svg_id, viewbox, points, initial_connections = []) {
    const svg = document.getElementById(svg_id)
    if (!svg) {
      console.error(`SVG with id "${svg_id}" not found`)
      return
    }
    
    svg.setAttribute('viewBox', viewbox)
    const connectionGroup = document.createElementNS("http://www.w3.org/2000/svg", "g")
    connectionGroup.classList.add("connectionGroup")
    const pointGroup = document.createElementNS("http://www.w3.org/2000/svg", "g")
    pointGroup.classList.add("pointGroup")
    svg.appendChild(connectionGroup)
    svg.appendChild(pointGroup)
    
    const state = {
      points: [],
      connections: [],
      active_point: null,
      initial_connections: initial_connections
    }
    
    // Create points
    points.forEach(([x, y, color]) => {
      const circle = document.createElementNS("http://www.w3.org/2000/svg", "circle")
      circle.setAttribute("cx", x)
      circle.setAttribute("cy", y)
      circle.setAttribute("r", "0.01")
      circle.classList.add("point", color)
      pointGroup.appendChild(circle)
      state.points.push({ element: circle, x, y, color })
    })
    
    svg_states.set(svg, state)
    svg.addEventListener('click', (e) => handle_click(svg, e))
    
    // Draw initial connections if any
    if (initial_connections.length > 0) {
      show_connections(svg_id, initial_connections)
    }
    
    return svg
  }

  function handle_click(svg, e) {
    const state = svg_states.get(svg)
    const rect = svg.getBoundingClientRect()
    const viewBox = svg.viewBox.baseVal
    
    // Transform into svg-coordinate-space
    const x = viewBox.x + (e.clientX - rect.left) * viewBox.width / rect.width
    const y = viewBox.y + (e.clientY - rect.top) * viewBox.height / rect.height
    
    // Try and get nearest point
    let nearest = null
    let min_dist = 0.1
    state.points.forEach(point => {
      const dist = Math.hypot(point.x - x, point.y - y)
      if (dist < min_dist) {
        min_dist = dist
        nearest = point
      }
    })
    
    // No nearest point, give up?
    if (!nearest) {
      deactivate_all(state)
      return
    }
    
    // Try and make a connection!
    if (!state.active_point) {
      activate_point(state, nearest)
    } else if (state.active_point === nearest) {
      deactivate_all(state)
    } else if (state.active_point.color !== nearest.color) {
      // Check if this exact connection already exists
      const existing_idx = state.connections.findIndex(conn =>
        (conn.point_a === state.active_point && conn.point_b === nearest) ||
        (conn.point_b === state.active_point && conn.point_a === nearest)
      )
      
      if (existing_idx !== -1) {
        // Remove existing connection
        const conn = state.connections[existing_idx]
        animate_out_and_remove(conn.element)
        state.connections.splice(existing_idx, 1)
        check_intersections(svg)
      } else {
        // Create new connection
        create_connection(svg, state.active_point, nearest)
      }
      deactivate_all(state)
    } else {
      activate_point(state, nearest)
    }
  }

  function activate_point(state, point) {
    state.points.forEach(p => p.element.classList.remove('active'))
    point.element.classList.add('active')
    state.active_point = point
  }

  function deactivate_all(state) {
    state.points.forEach(p => p.element.classList.remove('active'))
    state.active_point = null
  }

  function create_connection(svg, point_a, point_b, skip_animation = false) {
    const state = svg_states.get(svg)

    // Remove existing connections for both points
    remove_connections_for_points(svg, [point_a, point_b])

    const line = document.createElementNS("http://www.w3.org/2000/svg", "line")
    line.setAttribute("x1", point_a.x)
    line.setAttribute("y1", point_a.y)
    line.setAttribute("x2", point_b.x)
    line.setAttribute("y2", point_b.y)
    line.classList.add("connection")

    svg.querySelector(".connectionGroup").appendChild(line)

    if (!skip_animation) {
        const length = Math.hypot(point_b.x - point_a.x, point_b.y - point_a.y)
        line.style.strokeDasharray = `${length} ${length}`
        line.style.strokeDashoffset = length
        
        // Small delay ensures animation works
        setTimeout(() => {
        line.style.strokeDashoffset = 0
        }, 10)
    }

    state.connections.push({ element: line, point_a, point_b })
    check_intersections(svg)
  }

  function remove_connections_for_points(svg, points) {
    const state = svg_states.get(svg)
    const point_set = new Set(points)
    
    state.connections = state.connections.filter(conn => {
      if (point_set.has(conn.point_a) || point_set.has(conn.point_b)) {
        animate_out_and_remove(conn.element)
        return false
      }
      return true
    })
  }

  function animate_out_and_remove(element) {
    const length = Math.hypot(
      element.x2.baseVal.value - element.x1.baseVal.value,
      element.y2.baseVal.value - element.y1.baseVal.value
    )
    element.style.strokeDashoffset = length
    setTimeout(() => element.remove(), 500)
  }

  function check_intersections(svg) {
    const state = svg_states.get(svg)
    
    state.connections.forEach(c => c.element.classList.remove('invalid'))
    
    for (let i = 0; i < state.connections.length; i++) {
      for (let j = i + 1; j < state.connections.length; j++) {
        if (segments_intersect(state.connections[i], state.connections[j])) {
          state.connections[i].element.classList.add('invalid')
          state.connections[j].element.classList.add('invalid')
        }
      }
    }
  }

  function segments_intersect(conn_a, conn_b) {
    const a1 = { x: conn_a.point_a.x, y: conn_a.point_a.y }
    const a2 = { x: conn_a.point_b.x, y: conn_a.point_b.y }
    const b1 = { x: conn_b.point_a.x, y: conn_b.point_a.y }
    const b2 = { x: conn_b.point_b.x, y: conn_b.point_b.y }
    
    const det = (a2.x - a1.x) * (b2.y - b1.y) - (a2.y - a1.y) * (b2.x - b1.x)
    if (Math.abs(det) < 1e-10) return false
    
    const t = ((b1.x - a1.x) * (b2.y - b1.y) - (b1.y - a1.y) * (b2.x - b1.x)) / det
    const u = ((b1.x - a1.x) * (a2.y - a1.y) - (b1.y - a1.y) * (a2.x - a1.x)) / det
    
    return t > 0 && t < 1 && u > 0 && u < 1
  }

  function reset_svg(svg_id) {
    const svg = document.getElementById(svg_id)
    const state = svg_states.get(svg)
    if (!state) return
    
    state.connections.forEach(conn => animate_out_and_remove(conn.element))
    state.connections = []
    deactivate_all(state)
  }

  function show_connections(svg_id, connection_pairs) {
   const svg = document.getElementById(svg_id)
   const state = svg_states.get(svg)
   if (!state) return
   
   const find_point = (x, y) => 
     state.points.find(p => Math.abs(p.x - x) < 0.001 && Math.abs(p.y - y) < 0.001)
   
   // Helper to check if two connections are the same (regardless of direction)
   const same_connection = (conn, x1, y1, x2, y2) => {
     return (Math.abs(conn.point_a.x - x1) < 0.001 && 
             Math.abs(conn.point_a.y - y1) < 0.001 &&
             Math.abs(conn.point_b.x - x2) < 0.001 && 
             Math.abs(conn.point_b.y - y2) < 0.001) ||
            (Math.abs(conn.point_a.x - x2) < 0.001 && 
             Math.abs(conn.point_a.y - y2) < 0.001 &&
             Math.abs(conn.point_b.x - x1) < 0.001 && 
             Math.abs(conn.point_b.y - y1) < 0.001)
   }
   
   // Mark which connections should stay
   const to_keep = new Set()
   connection_pairs.forEach(([x1, y1, x2, y2]) => {
     state.connections.forEach((conn, idx) => {
       if (same_connection(conn, x1, y1, x2, y2)) {
         to_keep.add(idx)
       }
     })
   })
   
   // Remove connections not in target
   state.connections = state.connections.filter((conn, idx) => {
     if (to_keep.has(idx)) return true
     animate_out_and_remove(conn.element)
     return false
   })
   
   // Add missing connections
   connection_pairs.forEach(([x1, y1, x2, y2]) => {
     const already_exists = state.connections.some(conn => 
       same_connection(conn, x1, y1, x2, y2)
     )
     
     if (already_exists) return
     
     const point_a = find_point(x1, y1)
     const point_b = find_point(x2, y2)
     
     if (!point_a || !point_b) return
     
     const line = document.createElementNS("http://www.w3.org/2000/svg", "line")
     line.setAttribute("x1", point_a.x)
     line.setAttribute("y1", point_a.y)
     line.setAttribute("x2", point_b.x)
     line.setAttribute("y2", point_b.y)
     line.classList.add("connection")
     
     const length = Math.hypot(point_b.x - point_a.x, point_b.y - point_a.y)
     line.style.strokeDasharray = `${length} ${length}`
     line.style.strokeDashoffset = length
     
     svg.querySelector(".connectionGroup").appendChild(line)
     
     setTimeout(() => {
       line.style.strokeDashoffset = 0
     }, 10)
     
     state.connections.push({ element: line, point_a, point_b })
   })
   
   check_intersections(svg)
 }
</script>

# Crossing Guard

Futility Closet posed [the following puzzle](https://www.futilitycloset.com/2016/09/03/crossing-guard/):

> Suppose some $2n$ points in the plane are chosen so that no three are collinear, and then half are colored red and half blue. Will it always be possible to connect each red point with a blue one, in pairs, so that none of the connecting lines intersect?

"No three are collinear" means that no three points lie on a straight line. Although not specified, the "connection" between two points must be a straight line itself (otherwise it would always be easy to make such connections).

<svg id="example">
</svg>
<div class="center">
  <button id="examplePointsOnly">Show points only</button>
  <button id="exampleShowInvalid">Invalid solution</button>
  <button id="exampleShowValid">Valid solution</button>
</div>

<script>
{
  const svg = document.getElementById("example")
  const good_connections = [[0.54, 0.29, 0.73, 0.05],
  [0.82, 0.98, 0.53, 0.86],
  [0.41, 0.83, 0.44, 0.79],
  [0.37, 0.42, 0.08, 0.3 ],
  [0.1, 0.2, 0.35, 0.07],
  [0.05, 0.78, 0.05, 0.28],
  [0.76, 0.34, 0.87, 0.28],
  [0.18, 0.67, 0.67, 0.51],
  [0.16, 0.66, 0.47, 0.4 ],
  [0.81, 0.85, 0.28, 0.74]]
  const bad_connections = [[0.05, 0.78, 0.35, 0.07],
  [0.37, 0.42, 0.08, 0.3 ],
  [0.54, 0.29, 0.73, 0.05],
  [0.82, 0.98, 0.53, 0.86],
  [0.41, 0.83, 0.44, 0.79],
  [0.76, 0.34, 0.87, 0.28],
  [0.1, 0.2, 0.05, 0.28],
  [0.18, 0.67, 0.67, 0.51],
  [0.16, 0.66, 0.47, 0.4 ],
  [0.81, 0.85, 0.28, 0.74]]
  let points = []
  for (const [blue_x, blue_y, red_x, red_y] of good_connections) {
    points.push([blue_x, blue_y, "blue"])
    points.push([red_x, red_y, "red"])
  }

  create_interactive_svg("example", "0 0 1 1", points)

  document.getElementById("examplePointsOnly").addEventListener("click", () => {
    show_connections("example", [])
  })

  document.getElementById("exampleShowInvalid").addEventListener("click", () => {
    show_connections("example", bad_connections)
  })

  document.getElementById("exampleShowValid").addEventListener("click", () => {
    show_connections("example", good_connections)
  })
}
</script>

## Appproaches that don't work

Trying to provide all the connections at once, and proving that they do not intersect, seems difficult. Instead, I thought, it would help to only provide _one_ connection, and somehow prove that the puzzle remains possible.

Connecting the closest pair does not work:
<svg id="badClosest" class="small"></svg>
<script>
  create_interactive_svg("badClosest", "0 0.3 1 0.4", [[0.1, 0.5, "blue"], [0.5, 0.4, "blue"], [0.5, 0.6, "red"], [0.9, 0.5, "red"]], [[0.5,0.4, 0.5,0.6]])
</script>


Trying to connect an "outermost" pair does not work, i.e. a pair such that all other points are on only one side of the connection, because such a pair of points need not exist:
<svg id="badOuter" xmlns="http://www.w3.org/2000/svg" class="small"></svg>
<script>
  {
    let points = []
    const num_points = 3
    for (let i=0; i<num_points; i++) {
      const direction_x = Math.cos(2*Math.PI*(0.75+i/num_points))
      const direction_y = Math.sin(2*Math.PI*(0.75+i/num_points))
      points.push([0.5 + direction_x*0.35, 0.52 + direction_y*0.35, "blue"])
      points.push([0.5 + direction_x*0.45, 0.52 + direction_y*0.45, "red"])
    }
    create_interactive_svg("badOuter", "0 0 1 0.85", points)
  }
</script>

However, we need not put all points on one side of the connection. It would suffice to have a connection such that, on the "left" side of the connection, the number of blue and red points is equal, and on the "right" side of the connection, the number of blue and red points is equal. I considered starting with some fixed blue point, and somehow checking all the red points for possible connections. However, that need not work, either, consider the blue point in the center here:
<svg id="badLine" class="small">
<circle cx="0.5" cy="0.5" r="0.018" fill="none" style="stroke:#888; stroke-width:0.005; stroke-dasharray:0.005,0.005"/>
</svg>
<script>
create_interactive_svg("badLine", "0 0 1 0.85", [[0.5, 0.5, "blue"], [0.5, 0.15, "red"], [0.803, 0.675, "blue"], [0.197, 0.675, "red"]])
</script>

No matter which red point you connect the center blue point with, the remaining red and blue point will end up on different sides of the connection (however, you could still connect them without intersecting with the existing connection).

However, notice that if we connect the _outer_ blue point with a red point, _then_ the remaining points end up on the same side.
<svg id="betterLine" class="small">
</svg>
<script>
create_interactive_svg("betterLine", "0 0 1 0.85", [[0.5, 0.5, "blue"], [0.5, 0.15, "red"], [0.803, 0.675, "blue"], [0.197, 0.675, "red"]], [[0.803, 0.675, 0.5, 0.15]])
</script>

<!--

## A working approach
This last idea can be made to work. We can always find a pair of points such that, if we connect them, both sides of the connections are _balanced_: The number of blue points and red points on the "left" side of the connection is equal, and the number of blue points and red points on the "right" side of the connection is equal. Note that, because we assumed that no three points are collinear, all points are on either the left or the right side of the connection.


<svg viewbox="0 0 1 1" id="balancedLines" xmlns="http://www.w3.org/2000/svg">
</svg>
<script>
{
  const svg = document.getElementById("balancedLines")
  const balancedConnections = [[0, 6], [2, 7], [3, 5], [4, 1], [5, 5], [6, 4], [7, 15], [8, 2], [9, 9], [10, 14], [11, 9], [11, 10], [12, 4], [13, 0], [14, 11], [15, 3]]
  const redPoints = [[0.9406050357791532, 0.30217291120143713], [0.19920320003059871, 0.2761466456011049], [0.7834827372941037, 0.32314636565929133], [0.49072417266887736, 0.20744256756711438], [0.8468964675313825, 0.2015890165427336], [0.3625180215490413, 0.19950161371144526], [0.47090564885368763, 0.317252059770297], [0.6189387421466723, 0.2471649166544181], [0.904313208455598, 0.4443145183570353], [0.6800220816554267, 0.43668350637827336], [0.6532759366701174, 0.574994893245021], [0.740853013374749, 0.09240684229040436], [0.6577262549451347, 0.14462605097184855], [0.7975336324934315, 0.31413016587393117], [0.2293099584596482, 0.17799852551183776], [0.7451405368624647, 0.3862619733112171]]
  const bluePoints = [[0.7231752713243366, 0.5547656728233739], [0.787343523520555, 0.38197360851725837], [0.7224294811117647, 0.7214683895364428], [0.7215557228419519, 0.4496705811633159], [0.39660411196038814, 0.6349332955192127], [0.33145868459221767, 0.47631668813111083], [0.8677564959525109, 0.4291674975522184], [0.7463696109410137, 0.4623406602257559], [0.6322282973579857, 0.3712183579257316], [0.41171379776461015, 0.8501986023253991], [0.1115221854941684, 0.08959811406688026], [0.08528056296249104, 0.7095587478690922], [0.3983530552162603, 0.26334213093534137], [0.5645513023065665, 0.24485694670904185], [0.5283633497622343, 0.9145898874754527], [0.5059810093217636, 0.5159489164213025]]
  for (let point of bluePoints) {
    draw_point(svg, point, "blue")
  }
  for (let point of redPoints) {
    draw_point(svg, point, "red")
  }
}
</script>


<!-- TODO:
Make interactive.
Can we use CSS animations instead of stroke-dasharray hacks?
If not, is it worth the clunkiness to use animation-frames?
-->