---
title: Which Congressional Districts are Competitive
date: 2017-02-19 12:44:40
tags: Other
---
Hello World
{% raw %}
<style>

path {
  stroke-linejoin: round;
  stroke-linecap: round;
}

.districts {
  fill: #bbb;
}

.districts :hover {
  fill: orange;
}

.district-boundaries {
  pointer-events: none;
  fill: none;
  stroke: #fff;
  stroke-width: .5px;
}

.state-boundaries {
  pointer-events: none;
  fill: none;
  stroke: #fff;
  stroke-width: 1.5px;
}



</style>
<body>
<script src="https://d3js.org/d3.v3.min.js"></script>
<script src="https://d3js.org/queue.v1.min.js"></script>
<script src="https://d3js.org/topojson.v1.min.js"></script>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.1.1/jquery.min.js"></script>
<script>
$(function(){
var width = 960,
    height = 600;

var projection = d3.geo.albersUsa()
    .scale(960)
    .translate([width / 2, height / 2]);

var path = d3.geo.path()
    .projection(projection);

var svg = d3.select("div.entry").append("svg")
    .attr("width", width)
    .attr("height", height)    

queue()
    .defer(d3.json, "/files/us.json")
    .defer(d3.json, "/files/us-congress-113.json")
    .defer(d3.json, "/files/PVI.json")
    .await(ready);

function ready(error, us, congress, PVI) {
  if (error) throw error;

  svg.append("defs").append("path")
      .attr("id", "land")
      .datum(topojson.feature(us, us.objects.land))
      .attr("d", path);

  svg.append("clipPath")
      .attr("id", "clip-land")
    .append("use")
      .attr("xlink:href", "#land");
 
  svg.append("g")
      .attr("class", "districts")
      .attr("clip-path", "url(#clip-land)")
    .selectAll("path")
      .data(topojson.feature(congress, congress.objects.districts).features)     
    .enter().append("path")
      .attr("d", path)
       .attr("fill", function(d) {
          if(PVI[d.id] == null) {
              return '#bbb';
          }
          if(PVI[d.id].Flip === 'FALSE' && PVI[d.id].Current == 'R') {
            return '#FFE3EB';
          }
          if(PVI[d.id].Flip === 'FALSE' && PVI[d.id].Current == 'D') {
            return '#B5D3E7';
          }
          return '#663399';
      })
    .append("title")
      .text(function(d) { return d.id % 100; })
      

  svg.append("path")
      .attr("class", "district-boundaries")
      .datum(topojson.mesh(congress, congress.objects.districts, function(a, b) { return a !== b && (a.id / 1000 | 0) === (b.id / 1000 | 0); }))
      .attr("d", path);

  svg.append("path")
      .attr("class", "state-boundaries")
      .datum(topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; }))
      .attr("d", path);
}

});
</script>
{% endraw %}