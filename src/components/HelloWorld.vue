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
                          var labelZ;
                          if (point2GeoPosition.height >= point1GeoPosition.height) {
                            labelZ = point1GeoPosition.height + (point2GeoPosition.height - point1GeoPosition.height)/2.0;
                          } else {
                            labelZ = point2GeoPosition.height + (point1GeoPosition.height - point2GeoPosition.height)/2.0;
                          };
                          distanceLabel, horizontalLabel, verticalLabel = vueComponent.addPolylineWithLabel(point1, point2, labelZ, viewer, point1GeoPosition , point2GeoPosition, ellipsoid, horizontalLabel, distanceLabel, verticalLabel);
                          vueComponent.showButton = true;
                      }
                  }
              }
          }
      }, Cesium.ScreenSpaceEventType.LEFT_CLICK);
    },

    // add polyline & distance label from point 1 to point 2
    addPolylineWithLabel(point1, point2, height, viewer, point1GeoPosition, point2GeoPosition, ellipsoid, horizontalLabel, distanceLabel, verticalLabel) {
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
      var pointsLongitude = Cesium.Math.lerp(point1.position.x, point1.position.y, 0.5);
      var pointsLatitude = Cesium.Math.lerp(point2.position.x, point2.position.y, 0.5);
      console.log(pointsLongitude)
      // var pointsHeight = Cesium.Math.lerp(polylinePositions[0].height, polylinePositions[1].height, 0.5);
      var polylinePositions = [
          new Cesium.Cartographic.fromCartesian(point1.position),
          new Cesium.Cartographic.fromCartesian(point2.position)
      ];
      const promise = viewer.scene.sampleHeightMostDetailed(polylinePositions);
      promise.then(function(updatedPosition) {
        polylinePositions = updatedPosition
          // positions[0].height and positions[1].height have been updated.
          // updatedPositions is just a reference to positions.
      });
      var cartesian1 = new Cesium.Cartesian3.fromRadians(polylinePositions[0].longitude, polylinePositions[0].latitude, polylinePositions[0].height)
      var cartesian2 = new Cesium.Cartesian3.fromRadians(polylinePositions[1].longitude, polylinePositions[1].latitude, polylinePositions[1].height)
      console.log(cartesian1)
      console.log(cartesian2)
      const count = 30;
      const cartesians = new Array(count);
      for (let i = 0; i < count; ++i) {
        const offset = i / (count - 1);
        cartesians[i] = Cesium.Cartesian3.lerp(
          cartesian1,
          cartesian2,
          offset,
          new Cesium.Cartesian3()
        );
      }
      console.log(cartesians);
      viewer.scene.clampToHeightMostDetailed(cartesians)
      .then(function (clampedCartesians) {
        for (let i = 0; i < count; ++i) {
          viewer.entities.add({
            position: clampedCartesians[i],
            ellipsoid: {
              radii: new Cesium.Cartesian3(0.2, 0.2, 0.2),
              material: Cesium.Color.RED,
            },
          });
        }

        viewer.entities.add({
          polyline: {
            positions: clampedCartesians,
            arcType: Cesium.ArcType.NONE,
            width: 2,
            material: new Cesium.PolylineOutlineMaterialProperty({
              color: Cesium.Color.YELLOW,
            }),
            depthFailMaterial: new Cesium.PolylineOutlineMaterialProperty(
              {
                color: Cesium.Color.YELLOW,
              }
            ),
          },
        });
      });
      var pointsLongitude = Cesium.Math.lerp(polylinePositions[0].longitude, polylinePositions[1].longitude, 0.5);
      var pointsLatitude = Cesium.Math.lerp(polylinePositions[0].latitude, polylinePositions[1].latitude, 0.5);
      var pointsHeight = Cesium.Math.lerp(polylinePositions[0].height, polylinePositions[1].height, 0.5);
      this.details.push(polylinePositions[0]);
      this.details.push(polylinePositions[1]);
      this.details.push({
        'longitude': pointsLongitude,
        'latitude': pointsLatitude,
        'height': pointsHeight
      });
      // this.distance.push(["horizontal distance",this.getHorizontalDistanceString(point1, point2)])
      label.text = this.getDistanceString(point1, point2, polylinePositions[0], polylinePositions[1]);
      this.distance.push(["distance",label.text])
      distanceLabel = viewer.entities.add({
          position: this.getMidpoint(point1, point2, height),
          label: label
      });

      // if (!Cesium.Entity.supportsPolylinesOnTerrain(viewer.scene)) {
      //   window.alert(
      //     "Polylines on terrain are not supported on this platform"
      //   );
      // }
      // console.log(point1)
      // console.log(point2)
      // viewer.entities.add({
      //   polyline: {
      //     positions: Cesium.Cartesian3.fromDegreesArray([
      //       point1.longitude,
      //       point1.latitude,
      //       point2.longitude,
      //       point2.latitude,
      //     ]),
      //     clampToGround: true,
      //     width: 10,
      //     material: new Cesium.PolylineOutlineMaterialProperty({
      //       color: Cesium.Color.ORANGE,
      //       outlineWidth: 2,
      //       outlineColor: Cesium.Color.BLACK,
      //     }),
      //   },
      // });
      // this.distance.push(["vetical distance",this.getVerticalDistanceString(point1GeoPosition, point2GeoPosition)])
      // using Cesium.Math.lerp(p, q, time)
      // var pointsLongitude = Cesium.Math.lerp(polylinePositions[0].longitude, polylinePositions[1].longitude, 0.5);
      // var pointsLatitude = Cesium.Math.lerp(polylinePositions[0].latitude, polylinePositions[1].latitude, 0.5);
      // var pointsHeight = Cesium.Math.lerp(polylinePositions[0].height, polylinePositions[1].height, 0.5);
      // console.log(pointsLongitude);
      // console.log(pointsLatitude);
      // console.log(pointsHeight);
      // adding wall from point1 to point2
      var wallPosition = [
        new Cesium.Cartesian3.fromRadians(polylinePositions[0].longitude, polylinePositions[0].latitude, polylinePositions[0].height),
        new Cesium.Cartesian3.fromRadians(polylinePositions[1].longitude, polylinePositions[1].latitude, polylinePositions[1].height)
      ];
      // var wallPosition = new Cesium.Cartesian3.fromArray(cartesians);
      viewer.entities.add({
        name: "Blue wall at height",
        wall: {
          show: true,
          positions: wallPosition,
          minimumHeights: [20.0, 20.0],
          material: Cesium.Color.LIGHTGRAY,
        },
      });
      return distanceLabel, horizontalLabel, verticalLabel
    },
    // get horizontal distance from point 1 to point 2
    // getHorizontalDistanceString(point1, point2) {
    //   this.geodesic.setEndPoints(point1.cartographic, point2.cartographic);
    //   var meters = this.geodesic.surfaceDistance.toFixed(2);
    //   if (meters >= 1000) {
    //       return (meters / 1000).toFixed(1) + ' KM';
    //   }
    //   return meters + ' M';
    // },
    // get vertical distance from point 1 to point 2
    // getVerticalDistanceString(point1GeoPosition, point2GeoPosition) {
    //   var heights = [point1GeoPosition.height, point2GeoPosition.height];
    //   var meters = Math.max.apply(Math, heights) - Math.min.apply(Math, heights);
    //   if (meters >= 1000) {
    //       return (meters / 1000).toFixed(1) + ' KM';
    //   }
    //   return meters.toFixed(2) + ' M';
    // },
    // get actual distance from point 1 to point 2
    getDistanceString(point1, point2, polylinePosition1, polylinePosition2) {
      this.geodesic.setEndPoints(point1.cartographic, point2.cartographic);
      var horizontalMeters = this.geodesic.surfaceDistance.toFixed(2);
      var heights = [polylinePosition1.height, polylinePosition2.height];
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