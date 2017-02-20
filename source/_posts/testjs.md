---
title: Which Congressional Districts are Competitive
date: 2017-02-19 12:44:40
tags: Other
---
For the 2018 Elections, the districts in pink are those that are strongly Republican based on voter registration. The ones in light blue are strongly democratic. The ones in dark purple are the ones that are currently held by Republicans, but have a Partisan Voter Index (PVI) under 4, which means that with a good ground campaign, would be well within the realm of possibility to flip. There are a total of 33 nominally competitive seats which is sufficient to flip control of the House of Representatives from Republican to Democrat. 

-- Update

Blindly grabbing data from Wikipedia is dumb. Previous map had old data of the 114th Congress. Current map is of the 115th congress. Some updates are that I've added in the Representative's name, as well as listed the open seats due to special elections in Green. Montana I left as red even though it's technically open, because of Zinke is supposed to be appointed to be secretary of the interior.

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
var width = 800,
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
          if(PVI[d.id].Flip === 'OPEN') {
            return '#3f704d';
          }
          return '#663399';
      })
    .append("title")
      .text(function(d) { if(PVI[d.id] == null) {
          return 'Open\nDistrict: ' + (d.id % 100);
      } 
        return PVI[d.id].Rep +'\nDistrict: ' + (d.id % 100); })
      

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