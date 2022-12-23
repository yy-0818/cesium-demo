<script setup>
import * as Cesium from "cesium";
import { onMounted } from "vue";
onMounted(() => {
  Cesium.Ion.defaultAccessToken =
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJmMjc3NmUzNi1lZjg3LTRlMjYtYWJlYy0wMzkyOWY2Y2JiNzkiLCJpZCI6MTE5MDgyLCJpYXQiOjE2NzE3MjY4MjJ9.qc799X5fiB31WvOPAzn6GZeBELdStiVKoOQufYpBJME";
  var custom = new Cesium.ArcGisMapServerImageryProvider({
    // url: "https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer",
    url: "https://services.arcgisonline.com/ArcGIS/rest/services/World_Street_Map/MapServer",
  });
  var viewer = new Cesium.Viewer("cesiumContainer", {
    imageryProvider: custom, // 图像提供者
    animation: false, // 是否显示左下角动画控件
    fullscreenButton: false, // 是否显示全屏按钮
    geocoder: false, // 是否显示地名查找控件
    homeButton: false, // 是否显示home键
    navigationHelpButton: false, // 是否显示帮助信息控件
    sceneModePicker: false, // 是否显示投影方式
    timeline: false, // 是否显示时间线控件
    baseLayerPicker: false, // 是否显示图层选择控件
    selectionIndicator: false, // 是否显示指示器组件
    infoBox: false,

    // cesium地形
    terrainProvider: Cesium.createWorldTerrain({
      requestWatermark: true,
      requestVertexNormals: true,
      requestWaterMask: true, //增加水面特效
    }),
  });

  viewer._cesiumWidget._innerCreditContainer.style.display = "none"; // 去除logo信息
  viewer.scene.debugShowFramesPerSecond = false; // 显示帧速

  viewer.camera.setView({
    destination: new Cesium.Cartesian3(1332761, -4662399, 4137888),
    orientation: {
      heading: 0.6,
      pitch: -0.66,
    },
  });

  // 添加纽约3D建筑，75343为官方资产ID
  const city = viewer.scene.primitives.add(
    new Cesium.Cesium3DTileset({
      url: Cesium.IonResource.fromAssetId(75343),
    })
  );

  var transparentStyle = new Cesium.Cesium3DTileStyle({
    color: {
      // 根据条件判断具体的颜色
      conditions: [
        ["${Height} >= 300", "rgba(45,0,75,0.5)"],
        ["${Height} >= 200", "rgb(102,71,151)"],
        ["${Height} >= 100", "rgba(170,162,204,0.5)"],
        ["${Height} >= 50", "rgba(224,226,238,0.5)"],
        ["${Height} >= 25", "rgba(252,230,200,0.5)"],
        ["${Height} >= 10", "rgba(248,176,87,0.5)"],
        ["${Height} >= 5", "rgba(198,106,11,0.5)"],
        ["true", "rgb(127,59,8)"],
      ],
    },
  });
  city.style = transparentStyle;

  // var geojsonOptions = {
  //   clampToGround: false,
  // };

  // Geojson文件加载
  var neighborHoodsPromise = Cesium.GeoJsonDataSource.load(
    "./assets/SampleData/sampleNeighborhoods.geojson"
  );

  var neighborhoods;
  //当GeoJson文件加载成功后的操作
  neighborHoodsPromise.then((dataSource) => {
    // 将新数据作为实体添加到查看器
    viewer.dataSources.add(dataSource);
    //将实例保存到变量neighborhoods里
    neighborhoods = dataSource.entities;
    // 获取实例里的数据，是一个数组
    var neighborhoodEntities = dataSource.entities.values;
    //遍历这个数组以便对每一项进行样式配置，并且把这些数据放置到地形表面
    for (var i = 0; i < neighborhoodEntities.length; i++) {
      //每次循环到的数据都单独拿到
      var entity = neighborhoodEntities[i];
      //判断一下定义的多边形是否存在
      if (Cesium.defined(entity.polygon)) {
        // 使用kml邻域值作为实体名称
        entity.name = entity.properties.neighborhood;
        // 将多边形材质设置为随机的半透明颜色
        entity.polygon.material = Cesium.Color.fromRandom({
          red: 0.1,
          maximumGreen: 0.5,
          minimumBlue: 0.5,
          alpha: 0.6,
        });
        // 告诉多边形为地形着色。 ClassificationType.CESIUM_3D_TILE 将为 3D 图块集着色，而 ClassificationType.BOTH 将为 3d 图块和地形着色（BOTH 是默认值）
        entity.polygon.classificationType = Cesium.ClassificationType.TERRAIN;
        // 生成多边形中心,意思就是设置这个多边形的位置
        var polyPositions = entity.polygon.hierarchy.getValue(
          Cesium.JulianDate.now()
        ).positions;
        // 上面生成中心点，这里进行配置中心点
        var polyCenter = Cesium.BoundingSphere.fromPoints(polyPositions).center;
        // 设置在地球的表面
        polyCenter = Cesium.Ellipsoid.WGS84.scaleToGeodeticSurface(polyCenter);
        entity.position = polyCenter;
        // 生成每个多边形的标签
        entity.label = {
          text: entity.name,
          showBackground: true,
          scale: 0.6,
          //标签水平的位置
          horizontalOrigin: Cesium.HorizontalOrigin.CENTER,
          //标签垂直的位置
          verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
          //设置一下标签展示范围，摄像头高度
          distanceDisplayCondition: new Cesium.DistanceDisplayCondition(
            10.0,
            8000.0
          ),
          //超过这个范围标签就不能被点击
          disableDepthTestDistance: 100.0,
        };
      }
    }
  });

  //配置地标文件
  var kmlOptions = {
    camera: viewer.scene.camera,
    canvas: viewer.scene.canvas,
    // 如果我们想要将几何特征（多边形、线串和线性环）固定在地面上，则为 true。
    clampToGround: true,
  };
  var geocachePromise = Cesium.KmlDataSource.load(
    "./assets/SampleData/sampleGeocacheLocations.kml",
    kmlOptions
  );

  // 将 geocache 广告牌实体添加到场景中并为其设置样式
  geocachePromise.then(function (dataSource) {
    // 将新数据作为实体添加到查看器
    viewer.dataSources.add(dataSource);

    // 获取实体数组
    var geocacheEntities = dataSource.entities.values;

    for (var i = 0; i < geocacheEntities.length; i++) {
      var entity = geocacheEntities[i];
      if (Cesium.defined(entity.billboard)) {
        // 调整垂直原点，使图钉位于地形上
        entity.billboard.verticalOrigin = Cesium.VerticalOrigin.BOTTOM;
        entity.billboard.image = "/assets/tagpark.png";
        // 禁用标签以减少混乱
        entity.label = undefined;
        // 添加距离显示条件
        entity.billboard.distanceDisplayCondition =
          new Cesium.DistanceDisplayCondition(10.0, 20000.0);
        // 以度为单位计算纬度和经度
        var cartographicPosition = Cesium.Cartographic.fromCartesian(
          entity.position.getValue(Cesium.JulianDate.now())
        );
        var latitude = Cesium.Math.toDegrees(cartographicPosition.latitude);
        var longitude = Cesium.Math.toDegrees(cartographicPosition.longitude);
        // 修改描述
        var description =
          '<table class="cesium-infoBox-defaultTable cesium-infoBox-defaultTable-lighter"><tbody>' +
          "<tr><th>" +
          "Longitude" +
          "</th><td>" +
          longitude.toFixed(5) +
          "</td></tr>" +
          "<tr><th>" +
          "Latitude" +
          "</th><td>" +
          latitude.toFixed(5) +
          "</td></tr>" +
          "<tr><th>" +
          "实时人流" +
          "</th><td>" +
          Math.floor(Math.random() * 20000) +
          "</td></tr>" +
          "<tr><th>" +
          "安全等级" +
          "</th><td>" +
          Math.floor(Math.random() * 5) +
          "</td></tr>" +
          "</tbody></table>";
        entity.description = description;
      }
    }
  });

  // 从czml文件加载模型路径
  var dronePromise = Cesium.CzmlDataSource.load(
    "./assets/SampleData/sampleFlight.czml"
  );

  // 模型实体
  var drone;
  dronePromise.then(function (dataSource) {
    viewer.dataSources.add(dataSource);
    drone = dataSource.entities.getById("Aircraft/Aircraft1");
    drone.model = {
      uri: "./assets/SampleData/Models/ferrari2.gltf",
      minimumPixelSize: 100,
      maximumScale: 800,
      silhouetteColor: Cesium.Color.WHITE,
      silhouetteSize: 1,
    };

    drone.orientation = new Cesium.VelocityOrientationProperty(drone.position);
    drone.viewFrom = new Cesium.Cartesian3(0, -30, 30);
    viewer.clock.shouldAnimate = true;
  });
});
</script>

<template>
  <div id="cesiumContainer"></div>
</template>

<style>
html,
body,
#cesiumContainer {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
