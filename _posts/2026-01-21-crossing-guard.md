---
layout: default
title: Crossing Guard
---

<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
    max-width: 600px;
    margin: 2em auto;
    padding: 1em;
  }
  svg {
    width: 100%;
    height: auto;
    display: block;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  svg circle.point {
    cursor: pointer;
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
  svg circle.point.left-side {
    stroke: #333;
    stroke-width: 0.003;
    stroke-dasharray: 0.01 0.0025;
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
  svg line.hyperplane {
    stroke: #333;
    stroke-width: 0.003;
  }
  div.center {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5em;
  }
  .counter-row {
    display: flex;
    justify-content: space-around;
    font-size: 0.95em;
  }
  .counter.blue { color: #47A; }
  .counter.red { color: #E67; }
  .side-label {
    font-weight: 500;
  }
  span.left-decoration {
    text-decoration: underline dashed #333;
  }
   .controls {
    margin-top: 1em;
    display: flex;
    flex-direction: column;
    gap: 0.8em;
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

Try finding a solution for this example!

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

Finding a solution for this one example is not very difficult, but we want to prove that a solution exists for _every_ possible puzzle.

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
<circle cx="0.5" cy="0.5" r="0.018" fill="none" style="stroke:#888; stroke-width:0.005; stroke-dasharray:0.011,0.005"/>
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


## A working approach
This last idea can be made to work. We can always find a pair of points such that, if we connect them, both sides of the connections are _balanced_: The number of blue points and red points on the "left" side of the connection is equal, and the number of blue points and red points on the "right" side of the connection is equal. Note that, because we assumed that no three points are collinear, all points are on either the left or the right side of the connection.

<svg id="hyperplaneSvg" viewBox="0 0 1 1"></svg>
  
<div class="controls">
  <div class="counter-row">
    <div class="counter blue">
      <span class="left-decoration"><span class="side-label">Left:</span> Blue <span id="leftBlue">0</span> <span id="leftBlueCheck"></span></span>
    </div>
    <div class="counter blue">
      <span><span class="side-label">Right:</span> Blue <span id="rightBlue">0</span> <span id="rightBlueCheck"></span></span>
    </div>
  </div>
  <div class="counter-row">
    <div class="counter red">
      <span><span class="side-label left">Left:</span> Red <span id="leftRed">0</span> <span id="leftRedCheck"></span></span>
    </div>
    <div class="counter red">
      <span><span class="side-label">Right:</span> Red <span id="rightRed">0</span> <span id="rightRedCheck"></span></span>
    </div>
  </div>
  <button id="nextBalanced">Show balanced connection</button>
</div>

<script>
  const balancedConnections = [[0, 3], [0, 11], [0, 12], [0, 15], [1, 2], [2, 3], [3, 4], [4, 1], [5, 3], [6, 1], [6, 6], [8, 3], [9, 6], [10, 3], [11, 5], [14, 6]]
  const redPoints = [[0.782608810904449, 0.215383019741831], [0.7800764019647675, 0.34321759672754604], [0.4482963578302878, 0.1194697560035939], [0.1346633644037209, 0.3413103555183366], [0.06683586425825924, 0.1469515440770221], [0.526730924738229, 0.12669667710259205], [0.1671420453311267, 0.10234488131342523], [0.8117876709859181, 0.51438807398215], [0.34526739570414616, 0.13071143471103047], [0.5335816208094655, 0.4907968613477179], [0.41973036610977843, 0.07689066234646619], [0.2395345973740276, 0.13710186858765117], [0.6213605092948428, 0.8580360097424653], [0.5222192300191391, 0.8722032516094983], [0.07488338759084597, 0.6968420038822234], [0.2901125249129467, 0.5156133929065837]]
  const bluePoints = [[0.16222597070682504, 0.2415213861666461], [0.3528766468251009, 0.7190911151264505], [0.5635223690357274, 0.8796052472840002], [0.47552054717858266, 0.9414535581711121], [0.20158610547983447, 0.46187888986309517], [0.38440582609016, 0.6868172676443289], [0.348797495209178, 0.8848849311846316], [0.3589323384284045, 0.1735696100204765], [0.15636323363408117, 0.13344052006814722], [0.4542833121372101, 0.2908041071308217], [0.5226868628522884, 0.4275650971800061], [0.9459543781253772, 0.8454697921492772], [0.7916061810415032, 0.949803016592264], [0.4443406500187544, 0.18020287744500413], [0.5163289766719925, 0.31693087053044305], [0.5098857820635402, 0.9149993685686936]]
  let balancedIndex = 0 // Which balanced connection did we last display?
  
  const svg = document.getElementById('hyperplaneSvg')
  const state = {
    blueAnchor: bluePoints[0],
    redAnchor: redPoints[0],
    bluePoints: [],
    redPoints: [],
    currentLine: { x1: 0, y1: 0, x2: 1, y2: 1 },
    targetLine: { x1: 0, y1: 0, x2: 1, y2: 1 },
    animating: false
  }
  
  // Initialize SVG
  const hyperplaneGroup = document.createElementNS("http://www.w3.org/2000/svg", "g")
  const pointGroup = document.createElementNS("http://www.w3.org/2000/svg", "g")
  
  svg.appendChild(hyperplaneGroup)
  svg.appendChild(pointGroup)
  
  const hyperplane = document.createElementNS("http://www.w3.org/2000/svg", "line")
  hyperplane.classList.add("hyperplane")
  hyperplaneGroup.appendChild(hyperplane)
  
  // Create points
  bluePoints.forEach(([x, y], i) => {
    const circle = document.createElementNS("http://www.w3.org/2000/svg", "circle")
    circle.setAttribute("cx", x)
    circle.setAttribute("cy", y)
    circle.setAttribute("r", "0.01")
    circle.classList.add("point", "blue")
    pointGroup.appendChild(circle)
    state.bluePoints.push({ element: circle, x, y, index: i })
  })
  
  redPoints.forEach(([x, y], i) => {
    const circle = document.createElementNS("http://www.w3.org/2000/svg", "circle")
    circle.setAttribute("cx", x)
    circle.setAttribute("cy", y)
    circle.setAttribute("r", "0.01")
    circle.classList.add("point", "red")
    pointGroup.appendChild(circle)
    state.redPoints.push({ element: circle, x, y, index: i })
  })
  
  function updateHyperplane() {
    const [bx, by] = state.blueAnchor
    const [rx, ry] = state.redAnchor
    
    // Direction vector of the line
    const dx = rx - bx
    const dy = ry - by
    
    // Extend line to edges of viewbox
    const t = 10; // Large extension factor
    const x1 = bx - t * dx
    const y1 = by - t * dy
    const x2 = bx + t * dx
    const y2 = by + t * dy

    state.targetLine = { x1: x1, y1: y1, x2: x2, y2: y2 }

    renderHyperplane()

    if (!state.animating) {
      animateHyperplane();
    }
  }
  
  function pointSide(px, py) {
    // Use current line position (important during animation)
    const { x1, y1, x2, y2 } = state.currentLine

    if ((px == x1 && py == y1) || (px == x2 && py == y2)) return 'line'
    
    // Direction vector
    const dx = x2 - x1
    const dy = y2 - y1
    
    // Cross product to determine side
    const cross = dx * (py - y1) - dy * (px - x1)
    return cross > 0 ? 'left' : 'right'
  }
  
  function updateCounts() {
    let leftBlue = 0, leftRed = 0, rightBlue = 0, rightRed = 0
    
    state.bluePoints.forEach(p => {
      const side = pointSide(p.x, p.y)
      if (side === 'left') {
        leftBlue++
        p.element.classList.add("left-side")
      } else {
        p.element.classList.remove("left-side")
        if (side === 'right') rightBlue++
      }
    })
    
    state.redPoints.forEach(p => {
      const side = pointSide(p.x, p.y)
      if (side === 'left') {
        leftRed++
        p.element.classList.add("left-side")
      } else {
        p.element.classList.remove("left-side")
        if (side === 'right') rightRed++
      }
    })
    
    document.getElementById('leftBlue').textContent = leftBlue
    document.getElementById('leftRed').textContent = leftRed
    document.getElementById('rightBlue').textContent = rightBlue
    document.getElementById('rightRed').textContent = rightRed
    
    const leftBalanced = leftBlue === leftRed
    const rightBalanced = rightBlue === rightRed
    
    document.getElementById('leftBlueCheck').textContent = leftBalanced ? '✅' : '❌'
    document.getElementById('leftRedCheck').textContent = leftBalanced ? '✅' : '❌'
    document.getElementById('rightBlueCheck').textContent = rightBalanced ? '✅' : '❌'
    document.getElementById('rightRedCheck').textContent = rightBalanced ? '✅' : '❌'
  }
  
  function updateActivePoints() {
    // Remove all active classes
    [...state.bluePoints, ...state.redPoints].forEach(p => {
      p.element.classList.remove('active')
    })
    
    // Add active to current anchors
    state.bluePoints.forEach(p => {
      if (p.x === state.blueAnchor[0] && p.y === state.blueAnchor[1]) {
        p.element.classList.add('active')
      }
    })
    
    state.redPoints.forEach(p => {
      if (p.x === state.redAnchor[0] && p.y === state.redAnchor[1]) {
        p.element.classList.add('active')
      }
    })
  }
  
  svg.addEventListener('click', (e) => {
    const rect = svg.getBoundingClientRect()
    const viewBox = svg.viewBox.baseVal
    
    const x = viewBox.x + (e.clientX - rect.left) * viewBox.width / rect.width
    const y = viewBox.y + (e.clientY - rect.top) * viewBox.height / rect.height
    
    let nearestBlue = null
    let nearestRed = null
    let minDistBlue = 0.05
    let minDistRed = 0.05
    
    state.bluePoints.forEach(p => {
      const dist = Math.hypot(p.x - x, p.y - y)
      if (dist < minDistBlue) {
        minDistBlue = dist
        nearestBlue = p
      }
    })
    
    state.redPoints.forEach(p => {
      const dist = Math.hypot(p.x - x, p.y - y)
      if (dist < minDistRed) {
        minDistRed = dist
        nearestRed = p
      }
    })
    
    // Update whichever is closer
    if (nearestBlue && (!nearestRed || minDistBlue < minDistRed)) {
      state.blueAnchor = [nearestBlue.x, nearestBlue.y]
    } else if (nearestRed) {
      state.redAnchor = [nearestRed.x, nearestRed.y]
    }
    
    updateHyperplane()
    updateActivePoints()
  })

  function animateHyperplane() {
    const duration = 300
    const startTime = performance.now()
    const startLine = { ...state.currentLine }
    const targetLine = { ...state.targetLine }
    
    state.animating = true
    
    function animate(currentTime) {
      const elapsed = currentTime - startTime
      const progress = Math.min(elapsed / duration, 1)
      
      state.currentLine.x1 = startLine.x1 + (targetLine.x1 - startLine.x1) * progress
      state.currentLine.y1 = startLine.y1 + (targetLine.y1 - startLine.y1) * progress
      state.currentLine.x2 = startLine.x2 + (targetLine.x2 - startLine.x2) * progress
      state.currentLine.y2 = startLine.y2 + (targetLine.y2 - startLine.y2) * progress
      
      renderHyperplane()
      
      if (progress < 1) {
        requestAnimationFrame(animate)
      } else {
        state.animating = false
      }
    }
    
    requestAnimationFrame(animate)
  }
  function renderHyperplane() {
    hyperplane.setAttribute("x1", state.currentLine.x1)
    hyperplane.setAttribute("y1", state.currentLine.y1)
    hyperplane.setAttribute("x2", state.currentLine.x2)
    hyperplane.setAttribute("y2", state.currentLine.y2)
    
    updateCounts()
  }
  
  document.getElementById('nextBalanced').addEventListener('click', () => {
    const [redIdx, blueIdx] = balancedConnections[balancedIndex]
    state.blueAnchor = bluePoints[blueIdx]
    state.redAnchor = redPoints[redIdx]
    
    balancedIndex = (balancedIndex + 1) % balancedConnections.length
    
    updateHyperplane()
    updateActivePoints()
  })
  
  // Initial render
  updateHyperplane()
  updateActivePoints()
</script>

Finding balanced connections in this example is difficult: Only 6.25% (16/256) of the connections are balanced.