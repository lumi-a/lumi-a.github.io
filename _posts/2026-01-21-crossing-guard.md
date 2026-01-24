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


## Solutions that don't work

Trying to provide all the connections at once, and proving that they do not intersect, seems difficult. Instead, I thought, it would help to only provide _one_ connection, and somehow prove that the puzzle remains possible.

Connecting the closest pair does not work:
<svg viewbox="0 0.3 1 0.4" id="badClosest" xmlns="http://www.w3.org/2000/svg"></svg>
<script>
{
  const svg = document.getElementById("badClosest")
  draw_point(svg, [0.1, 0.5], "blue")
  draw_point(svg, [0.5, 0.4], "blue")
  draw_point(svg, [0.5, 0.6], "red")
  draw_point(svg, [0.9, 0.5], "red")
  draw_connection(svg, [0.5, 0.4], [0.5, 0.6])
}
</script>

