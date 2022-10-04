<!-- For Code Clean-up -->
<template>
  <div id="container">
    <div id="menu" style="padding-bottom: 5px">
      <span>VUE 3 + CESIUM</span>
      <!-- export csv button -->
      <button style="position:absolute; right: 10px" v-on:click="exportCSV">
        Export CSV
      </button>
      <!-- export dwg button -->
      <!-- <button @click="exportDWG">
        Export DWG
      </button> -->
    </div>
  </div>
</template>

<script>
export default {
  name: 'vue3.0_cesium_project',
  data() {
    return {
      geodesic: new Cesium.EllipsoidGeodesic(),
      details: [],
      distance: [],
      dataSource: null,
    };
  },
  async mounted() {
    this.load3DTileset();
  },
  methods: {
    load3DTileset() {
      // cesium ion token (should be added in env file or protected file for security)
      Cesium.Ion.defaultAccessToken = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI1ZjU3ZWUzMS0yMTJiLTRmNjUtYjE1YS0yYTU2MjQwOWEwZGIiLCJpZCI6MTA5OTA0LCJpYXQiOjE2NjQ3Mjc0Njh9.jYUr3jpVl2YzqB4MY1A0-KtWqxnp6sHcbyBdMH_RECc";
      var viewer = new Cesium.Viewer('container', {
        // the following options is for self define button's 
        // show/don't show in the viewer
        geocoder: false,
        homeButton: false,
        sceneModePicker: false,
        baseLayerPicker: false,
        navigationHelpButton: true,
        animation: false,
        timeline: false,
        fullscreenButton: true,
        vrButton: false,
        shadows: false,
        infoBox: false,
        CreditsDisplay: false,
        terrainProvider: new Cesium.CesiumTerrainProvider({
          url: Cesium.IonResource.fromAssetId(1),
        }),
      });
      viewer.scene.globe.depthTestAgainstTerrain = true;
      var camera = viewer.camera;
      var scene = viewer.scene;
      var ellipsoid = Cesium.Ellipsoid.WGS84;
      var tm_url = Cesium.IonResource.fromAssetId(1340967);
      var tm = new Cesium.Cesium3DTileset({
        url: tm_url,
      })
      var tileset = scene.primitives.add(tm);
      // remove the logo
      viewer._cesiumWidget._creditContainer.style.display = "none";
      tileset.readyPromise.then(function() {
        var boundingSphere = tileset.boundingSphere;
        camera.viewBoundingSphere(boundingSphere, new Cesium.HeadingPitchRange(0.5, -0.2, boundingSphere.radius * 4.0));
        camera.lookAtTransform(Cesium.Matrix4.IDENTITY);
        viewer.zoomTo(tileset);
      }).otherwise(function(error) {
        throw(error);
      });
      tileset.allTilesLoaded.addEventListener(function() {
        console.log('All tiles are loaded');
      });
      //  initialize additional variables for points, etc.
      var points = scene.primitives.add(new Cesium.PointPrimitiveCollection());
      var point1, point2;
      var point1GeoPosition, point2GeoPosition;
      var polylines = scene.primitives.add(new Cesium.PolylineCollection());
      var polyline1, polyline2, polyline3;
      var distanceLabel, verticalLabel, horizontalLabel;
      var LINEPOINTCOLOR = Cesium.Color.RED;
      var vueComponent = this;
      // handler for screen action inside the cesium container
      var handler = new Cesium.ScreenSpaceEventHandler(scene.canvas);
      handler.setInputAction(function(click) {
          if (scene.mode !== Cesium.SceneMode.MORPHING) {
              var pickedObject = scene.pick(click.position);
              if (scene.pickPositionSupported && Cesium.defined(pickedObject)) {
                  var cartesian = viewer.scene.pickPosition(click.position);
                  if (Cesium.defined(cartesian)) {
                      if (points.length === 2) {
                          points.removeAll();
                          polylines.removeAll();
                          viewer.entities.removeAll();
                          vueComponent.showButton = false;
                          vueComponent.details = [];
                          vueComponent.distance = [];
                      }
                      //add first point
                      if (points.length === 0) {
                          point1 = points.add({
                              position : new Cesium.Cartesian3(cartesian.x, cartesian.y, cartesian.z),
                              color : LINEPOINTCOLOR
                          });
                      } //add second point and lines
                      else if (points.length === 1) {
                          point2 = points.add({
                              position : new Cesium.Cartesian3(cartesian.x, cartesian.y, cartesian.z),
                              color : LINEPOINTCOLOR
                          }); 
                          point1GeoPosition = Cesium.Cartographic.fromCartesian(point1.position);
                          point2GeoPosition = Cesium.Cartographic.fromCartesian(point2.position);
                          // point3GeoPosition = Cesium.Cartographic.fromCartesian(new Cesium.Cartesian3(point2.position.x, point2.position.y, point1.position.z));  
                          // initialize position for the polyline
                          var pl1Positions = [
                            new Cesium.Cartesian3.fromRadians(point1GeoPosition.longitude, point1GeoPosition.latitude, point1GeoPosition.height),
                            new Cesium.Cartesian3.fromRadians(point2GeoPosition.longitude, point2GeoPosition.latitude, point2GeoPosition.height)
                          ];
                          // var pl2Positions = [
                          //   new Cesium.Cartesian3.fromRadians(point2GeoPosition.longitude, point2GeoPosition.latitude, point2GeoPosition.height),
                          //   new Cesium.Cartesian3.fromRadians(point2GeoPosition.longitude, point2GeoPosition.latitude, point1GeoPosition.height)
                          // ];
                          // var pl3Positions = [
                          //   new Cesium.Cartesian3.fromRadians(point1GeoPosition.longitude, point1GeoPosition.latitude, point1GeoPosition.height),
                          //   new Cesium.Cartesian3.fromRadians(point2GeoPosition.longitude, point2GeoPosition.latitude, point1GeoPosition.height)
                          // ];
                          // add polyline from point 1 to point 2
                          polyline1 = polylines.add({
                            show : true,
                            positions : pl1Positions,
                            width : 1,
                            material: new Cesium.Material({
                                fabric : {
                                    type : 'Color',
                                    uniforms : {
                                        color : LINEPOINTCOLOR
                                    }
                                }
                            })
                          }); 
                          // polyline2 = polylines.add({
                          //   show : true,
                          //   positions : pl2Positions,
                          //   width : 1,
                          //   material: new Cesium.Material({
                          //       fabric : {
                          //           type : 'PolylineDash',
                          //           uniforms : {
                          //               color : LINEPOINTCOLOR,
                          //           }
                          //       },
                          //   })
                          // });
                          // polyline3 = polylines.add({
                          //   show : true,
                          //   positions : pl3Positions,
                          //   width : 1,
                          //   material: new Cesium.Material({
                          //       fabric : {
                          //           type : 'PolylineDash',
                          //           uniforms : {
                          //               color : LINEPOINTCOLOR,
                          //           }
                          //       },
                          //   })
                          // }); 
                          var labelZ;
                          if (point2GeoPosition.height >= point1GeoPosition.height) {
                            labelZ = point1GeoPosition.height + (point2GeoPosition.height - point1GeoPosition.height)/2.0;
                          } else {
                            labelZ = point2GeoPosition.height + (point1GeoPosition.height - point2GeoPosition.height)/2.0;
                          };
                          distanceLabel, horizontalLabel, verticalLabel = vueComponent.addDistanceLabel(point1, point2, labelZ, viewer, point1GeoPosition , point2GeoPosition, ellipsoid, horizontalLabel, distanceLabel, verticalLabel);
                          vueComponent.showButton = true;
                      }
                  }
              }
          }
      }, Cesium.ScreenSpaceEventType.LEFT_CLICK);
    },

    // add distance label from point 1 to point 2
    addDistanceLabel(point1, point2, height, viewer, point1GeoPosition, point2GeoPosition, ellipsoid, horizontalLabel, distanceLabel, verticalLabel) {
      var label = {
        font : '14px monospace',
        showBackground : true,
        horizontalOrigin : Cesium.HorizontalOrigin.CENTER,
        verticalOrigin : Cesium.VerticalOrigin.CENTER,
        pixelOffset : new Cesium.Cartesian2(0, 0),
        eyeOffset: new Cesium.Cartesian3(0,0,-50),
        fillColor: Cesium.Color.WHITE,
      };
      point1.cartographic = ellipsoid.cartesianToCartographic(point1.position);
      point2.cartographic = ellipsoid.cartesianToCartographic(point2.position);
      point1.longitude = parseFloat(Cesium.Math.toDegrees(point1.position.x));
      point1.latitude = parseFloat(Cesium.Math.toDegrees(point1.position.y));
      point2.longitude = parseFloat(Cesium.Math.toDegrees(point2.position.x));
      point2.latitude = parseFloat(Cesium.Math.toDegrees(point2.position.y));
      this.details.push(point1GeoPosition);
      this.details.push(point2GeoPosition);
      label.text = this.getHorizontalDistanceString(point1, point2);
      console.log(point1GeoPosition);
      console.log(point2GeoPosition);
      console.log(point1);
      this.distance.push(["horizontal distance",label.text])
      // horizontalLabel = viewer.entities.add({
      //     position: this.getMidpoint(point1, point2, point1GeoPosition.height),
      //     label: label
      // });
      label.text = this.getDistanceString(point1, point2, point1GeoPosition, point2GeoPosition);
      this.distance.push(["points distance",label.text])
      distanceLabel = viewer.entities.add({
          position: this.getMidpoint(point1, point2, height),
          label: label
      });
      label.text = this.getVerticalDistanceString(point1GeoPosition, point2GeoPosition);
      this.distance.push(["vetical distance",label.text])
      // verticalLabel = viewer.entities.add({
      //     position: this.getMidpoint(point2, point2, height),
      //     label: label
      // });
      
      // adding wall from point1 to point2
      var wallPosition = [
        new Cesium.Cartesian3.fromRadians(point1GeoPosition.longitude, point1GeoPosition.latitude, point1GeoPosition.height),
        new Cesium.Cartesian3.fromRadians(point2GeoPosition.longitude, point2GeoPosition.latitude, point2GeoPosition.height)
      ];
      viewer.entities.add({
        name: "Blue wall at height",
        wall: {
          show: true,
          positions: wallPosition,
          minimumHeights: [20.0, 20.0],
          material: Cesium.Color.BLUE,
        },
      });
      this.dataSource = viewer;
      return distanceLabel, horizontalLabel, verticalLabel
    },
    // get horizontal distance from point 1 to point 2
    getHorizontalDistanceString(point1, point2) {
      this.geodesic.setEndPoints(point1.cartographic, point2.cartographic);
      var meters = this.geodesic.surfaceDistance.toFixed(2);
      if (meters >= 1000) {
          return (meters / 1000).toFixed(1) + ' KM';
      }
      return meters + ' M';
    },
    // get vertical distance from point 1 to point 2
    getVerticalDistanceString(point1GeoPosition, point2GeoPosition) {
      var heights = [point1GeoPosition.height, point2GeoPosition.height];
      var meters = Math.max.apply(Math, heights) - Math.min.apply(Math, heights);
      if (meters >= 1000) {
          return (meters / 1000).toFixed(1) + ' KM';
      }
      return meters.toFixed(2) + ' M';
    },
    // get actual distance from point 1 to point 2
    getDistanceString(point1, point2, point1GeoPosition, point2GeoPosition) {
      this.geodesic.setEndPoints(point1.cartographic, point2.cartographic);
      var horizontalMeters = this.geodesic.surfaceDistance.toFixed(2);
      var heights = [point1GeoPosition.height, point2GeoPosition.height];
      var verticalMeters = Math.max.apply(Math, heights) - Math.min.apply(Math, heights);
      var meters = Math.pow((Math.pow(horizontalMeters, 2) + Math.pow(verticalMeters, 2)), 0.5);

      if (meters >= 1000) {
          return (meters / 1000).toFixed(1) + ' KM';
      }
      return meters.toFixed(2) + ' M';
    },
    // get midpoint for label position
    getMidpoint(point1, point2, height) {
      var scratch = new Cesium.Cartographic();
      this.geodesic.setEndPoints(point1.cartographic, point2.cartographic);
      var midpointCartographic = this.geodesic.interpolateUsingFraction(0.5, scratch);
      return Cesium.Cartesian3.fromRadians(midpointCartographic.longitude, midpointCartographic.latitude, height);
    },  
    // save data to string & download as csv file
    exportCSV() {
      if (this.details.length != 0) {
        var csvString = [
          [
            "",
            "latitude",
            "longitude",
            "height"
          ],
          ...this.details.map(item => [
            "",
            item.latitude,
            item.longitude,
            item.height
          ])
        ]
        .map(e => e.join(",")) 
        .join("\n");
        csvString = csvString+ "\n" + this.distance.map(e => e.join(",")).join("\n");
        console.log(csvString)
        var blob = new Blob([csvString], { type: 'application/vnd.ms-excel' });
        const url = window.URL.createObjectURL(blob);
        var link = document.createElement('a');
        link.href = url;
        link.setAttribute('download', 'results.csv');
        document.body.appendChild(link);
        link.click();
        link.remove();
        window.URL.revokeObjectURL(url);
      }
    }
  },
}
</script>

<style scoped>
#container {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
</style>