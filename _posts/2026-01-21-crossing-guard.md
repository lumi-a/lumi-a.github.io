---
layout: default
title: Crossing Guard
---

# Crossing Guard

Futility Closet posed [the following puzzle](https://www.futilitycloset.com/2016/09/03/crossing-guard/):

> Suppose some $2n$ points in the plane are chosen so that no three are collinear, and then half are colored red and half blue. Will it always be possible to connect each red point with a blue one, in pairs, so that none of the connecting lines intersect?

"No three are collinear" means that no three points lie on a straight line. Although not specified, the "connection" between two points must be a straight line itself (otherwise it would always be easy to make such connections).

<style>
  svg {
    max-width: 500px;
    width: 100%;
    height: auto;
    display: block;
    margin: 2em auto;
    border: 1px solid #ccc;
    border-radius: 5px;
  }
  svg.small {
    max-width: 375px;
  }
  svg.tiny {
    max-width: 250px;
  }
  svg circle.point.red {
    fill: #EE6677; /* // https://sronpersonalpages.nl/~pault/ */
  }
  svg circle.point.blue {
    fill: #4477AA; 
  }
  svg line.connection {
    stroke: #888;
    stroke-width: 0.005;
    stroke-dasharray: 1.5 1.5;
    transition: stroke-dashoffset .5s;
  }
  svg:not(.showValid) line.goodHide {
    stroke-dashoffset: 1.5;
  }
  svg.showInvalid line.invalid {
    stroke: red;
  }
  svg:not(.showInvalid) line.badHide {
    stroke-dashoffset: 1.5;
  }
  div.center {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 0.5em;
  }
</style>
<svg viewbox="0 0 1 1" id="example" xmlns="http://www.w3.org/2000/svg">
</svg>
<div class="center">
  <button id="examplePointsOnly">Show points only</button>
  <button id="exampleShowInvalid">Invalid solution</button>
  <button id="exampleShowValid">Valid solution</button>
</div>

<script>
function draw_point(svg, coordinates, blue_or_red) {
  const point = document.createElementNS("http://www.w3.org/2000/svg", "circle")
  point.setAttribute("cx", `${coordinates[0]}`)
  point.setAttribute("cy", `${coordinates[1]}`)
  point.setAttribute("r", "0.01")
  point.classList.add("point")
  point.classList.add(blue_or_red)
  svg.appendChild(point)
}
function draw_connection(svg, point_a, point_b, classes = ["alwaysVisible"]) {
  const connection = document.createElementNS("http://www.w3.org/2000/svg", "line")
  connection.setAttribute("x1", `${point_a[0]}`)
  connection.setAttribute("y1", `${point_a[1]}`)
  connection.setAttribute("x2", `${point_b[0]}`)
  connection.setAttribute("y2", `${point_b[1]}`)
  connection.classList.add("connection")
  connection.classList.add(...classes)
  svg.appendChild(connection)
}
{
  const svg = document.getElementById("example")
  const points = [[[0.54, 0.29], [0.73, 0.05]],
  [[0.82, 0.98], [0.53, 0.86]],
  [[0.41, 0.83], [0.44, 0.79]],
  [[0.37, 0.42], [0.08, 0.3 ]],
  [[0.1, 0.2], [0.35, 0.07]],
  [[0.05, 0.78], [0.05, 0.28]],
  [[0.76, 0.34], [0.87, 0.28]],
  [[0.18, 0.67], [0.67, 0.51]],
  [[0.16, 0.66], [0.47, 0.4 ]],
  [[0.81, 0.85], [0.28, 0.74]]]
  const bad_points = [[[0.05, 0.78], [0.35, 0.07]],
  [[0.37, 0.42], [0.08, 0.3 ]],
  [[0.54, 0.29], [0.73, 0.05]],
  [[0.82, 0.98], [0.53, 0.86]],
  [[0.41, 0.83], [0.44, 0.79]],
  [[0.76, 0.34], [0.87, 0.28]],
  [[0.1, 0.2], [0.05, 0.28]],
  [[0.18, 0.67], [0.67, 0.51]],
  [[0.16, 0.66], [0.47, 0.4 ]],
  [[0.81, 0.85], [0.28, 0.74]]]

  // Draw good connections
  for (const point of points) {
    draw_connection(svg, point[0], point[1], ["goodHide"])
  }
  // Draw bad connections, draw red lines first
  draw_connection(svg, bad_points[0][0], bad_points[0][1], ["badHide", "invalid"])
  for (const point of bad_points.slice(1)) {
    draw_connection(svg, point[0], point[1], ["badHide"])
  }
  // Draw points on top
  for (const point of points) {
    // doesn't really matter which one is blue and which one is red
    draw_point(svg, point[0], "blue")
    draw_point(svg, point[1], "red")
  }
  document.getElementById("examplePointsOnly").addEventListener("click", () => {
    svg.classList.remove("showValid")
    svg.classList.remove("showInvalid")
  })
  document.getElementById("exampleShowInvalid").addEventListener("click", () => {
    svg.classList.remove("showValid")
    svg.classList.add("showInvalid")
  })
  document.getElementById("exampleShowValid").addEventListener("click", () => {
    svg.classList.add("showValid")
    svg.classList.remove("showInvalid")
  })
}
</script>


## Appproaches that don't work

Trying to provide all the connections at once, and proving that they do not intersect, seems difficult. Instead, I thought, it would help to only provide _one_ connection, and somehow prove that the puzzle remains possible.

Connecting the closest pair does not work:
<svg viewbox="0 0.3 1 0.4" id="badClosest" xmlns="http://www.w3.org/2000/svg" class="small"></svg>
<script>
{
  const svg = document.getElementById("badClosest")
  draw_connection(svg, [0.5, 0.4], [0.5, 0.6])
  draw_point(svg, [0.1, 0.5], "blue")
  draw_point(svg, [0.5, 0.4], "blue")
  draw_point(svg, [0.5, 0.6], "red")
  draw_point(svg, [0.9, 0.5], "red")
}
</script>

Trying to connect an "outermost" pair does not work, i.e. a pair such that all other points are on only one side of the connection, because such a pair of points need not exist:
<svg viewbox="0 0 1 0.85" id="badOuter" xmlns="http://www.w3.org/2000/svg" class="small"></svg>
<script>
{
  const svg = document.getElementById("badOuter")
  const num_points = 3
  for (let i=0; i<num_points; i++) {
    const direction_x = Math.cos(2*Math.PI*(0.75+i/num_points))
    const direction_y = Math.sin(2*Math.PI*(0.75+i/num_points))
    draw_point(svg, [0.5 + direction_x*0.35, 0.52 + direction_y*0.35], "blue")
    draw_point(svg, [0.5 + direction_x*0.45, 0.52 + direction_y*0.45], "red")
  }
}
</script>

However, we need not put all points on one side of the connection. It would suffice to have a connection such that, on the "left" side of the connection, the number of blue and red points is equal, and on the "right" side of the connection, the number of blue and red points is equal. I considered starting with some fixed blue point, and somehow checking all the red points for possible connections. However, that need not work, either, consider the blue point in the center here:
<svg viewbox="0 0 1 0.85" id="badLine" xmlns="http://www.w3.org/2000/svg" class="small">
<circle cx="0.5" cy="0.5" r="0.018" fill="none" style="stroke:#888; stroke-width:0.005"/>
</svg>
<script>
{
  const svg = document.getElementById("badLine")
  draw_point(svg, [0.5, 0.5], "blue")
  for (let i=0; i<3; i++) {
    const direction_x = Math.cos(2*Math.PI*(0.75+i/3))
    const direction_y = Math.sin(2*Math.PI*(0.75+i/3))
    draw_point(svg, [0.5 + direction_x*0.35, 0.5 + direction_y*0.35], i==0 ? "blue" : "red")
  }
}
</script>
No matter which red point you connect the center blue point with, the remaining red and blue point will end up on different sides of the connection (however, you could still connect them without intersecting with the existing connection).

However, notice that if we connect an _outer_ blue point with a red point, _then_ the remaining points end up on the same side.
<svg viewbox="0 0 1 0.85" id="betterLine" xmlns="http://www.w3.org/2000/svg" class="small">
<circle cx="0.5" cy="0.5" r="0.018" fill="none" style="stroke:#888; stroke-width:0.005"/>
</svg>
<script>
{
  const svg = document.getElementById("betterLine")
  connections = []
  for (let i of [0, 1]) {
    const direction_x = Math.cos(2*Math.PI*(0.75+i/3))
    const direction_y = Math.sin(2*Math.PI*(0.75+i/3))
    connections.push([0.5 + direction_x*0.35, 0.5 + direction_y*0.35])
  }
  draw_connection(svg, connections[0], connections[1])
  draw_point(svg, [0.5, 0.5], "blue")
  for (let i=0; i<3; i++) {
    const direction_x = Math.cos(2*Math.PI*(0.75+i/3))
    const direction_y = Math.sin(2*Math.PI*(0.75+i/3))
    draw_point(svg, [0.5 + direction_x*0.35, 0.5 + direction_y*0.35], i==0 ? "blue" : "red")
  }
}
</script>

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
TODO: Make this interactive.