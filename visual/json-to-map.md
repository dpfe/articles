豆皮粉儿们，大家好呀，今天这一期，由字节跳动数据平台的「小东」同学给大家带聊聊从GeoJSON/TopoJSON到地图绘制。

正文共： 3335字 14图

预计阅读时间： 9分钟

欢迎加入我们团队，后台回复内推，「豆皮范儿」后台回复加群，欢迎咨询和交流。

![加入我们团队](https://cdn.nlark.com/yuque/0/2021/png/276016/1630985148011-e6b92f21-a241-4dc1-9379-ce3d48027ca0.png?date=1630985149439)


这是一张再熟悉不过的地图，但右上角黑龙江与内蒙古之间的那片飞地是什么呢？

![image.png](https://tech-proxy.bytedance.net/tos/images/1585641053229_d4f8d2892c8e95d14b760464a931a321.png)

查一下，摘自互联网：

> 大兴安岭是黑龙江省下辖的一个非常特殊的地级行政单位，大兴安岭行政公署和大兴安岭林业集团公司实行政企合一的管理体制。行政公署为省政府派驻机构，所辖加格达奇和松岭两区，地权属于内蒙古自治区呼伦贝尔市。黑龙江每年向内蒙古支付租金，加格达奇属于“租界”。所以，加格达奇人是黑龙江的，地是内蒙古的，树是林业部的。

哦豁，原来如此。知道了，但是我突然更好奇这个地图是怎么画在网页上的呢？

---

将地图画在页面上，概括一下分三步。

1. 拿到地图信息数据
2. 投影
3. 画

## 地图数据

这里介绍两种可以用于表示地理信息的数据类型

- GeoJSON
- TopoJSON

#### GeoJSON

> GeoJSON 是一种基于 JSON 的地理空间数据交换格式，它定义了几种类型 JSON 对象以及它们组合在一起的方法，以表示有关地理要素、属性和它们的空间范围的数据。

如下表所示，坐标表示点。点连成线，线连成面，从而表现出各种各样的形状。

![image.png](https://tech-proxy.bytedance.net/tos/images/1585641114592_490e70a46c597ce4b52ddb1e007c1c68.png)

![image.png](https://tech-proxy.bytedance.net/tos/images/1585641176735_7ab84eb8388a13ffe56579347bf1f61e.png)
![image.png](https://tech-proxy.bytedance.net/tos/images/1585641204859_7c3523f58423e4fc6598d3efb2650305.png)
![image.png](https://tech-proxy.bytedance.net/tos/images/1585641231435_5d1aed537d861a695007c6e24ebbff17.png)

一段 GeoJSON 示例

```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [102.0, 0.5]
      },
      "properties": {
        "prop0": "value0"
      }
    },
    {
      "type": "Feature",
      "geometry": {
        "type": "LineString",
        "coordinates": [
          [102.0, 0.0],
          [103.0, 1.0],
          [104.0, 0.0],
          [105.0, 1.0]
        ]
      },
      "properties": {
        "prop0": "value0",
        "prop1": 0.0
      }
    },
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [100.0, 0.0],
            [101.0, 0.0],
            [101.0, 1.0],
            [100.0, 1.0],
            [100.0, 0.0]
          ]
        ]
      },
      "properties": {
        "prop0": "value0",
        "prop1": { "this": "that" }
      }
    }
  ]
}
```

#### TopoJSON

TopoJSON 是 GeoJSON 按拓扑学编码后的扩展形式，是由 D3 的作者 Mike Bostock 制定的。相比 GeoJSON 直接使用 Polygon、Point 之类的几何体来表示图形的方法，TopoJSON 中的每一个几何体都是通过将共享边（被称为 arcs）整合后组成的。

简单说，它的线并不是由点构成，而是有自己的 id。

边在一起组成几何形状时，最后一个坐标必须与后续边的第一个坐标（如果有）相同。例如，如果边 6 代表点序列 A→B→C，边 7 代表点序列 C→D→E，则[6，7]代表点序列 A→B→C→D→E。

由于边是共享的所以在表示不同方向时需要反转。使用补码表示。 -1（〜0）为反 0，-2（〜1）为反 1，依此类推。

![image](https://tech-proxy.bytedance.net/tos/images/1585641259750_300910ec23015c255d6f955d52a755b8)

以上图举例说明。
绿色点与红色点坐标分别代表两个点。
紫色边为 3。
如果定义蓝色边顺时针方向为 1，黑色边顺时针方向为 0，则蓝色边与黑色边组成几何图形为[[0, 1]]。
在此基础上红色顺时针为 2，则红色边与逆黑色边（~0 = -1）组成图形为[[2, -1]]。

#### 获取数据

1. http://datav.aliyun.com/tools/atlas/ 直接下载中国及省市 GeoJSON
2. http://www.naturalearthdata.com/ 获取各种各样的 shape 格式(.shp)地图数据，使用 mapshaper(https://github.com/mbloch/mapshaper)进行处理后导出为GeoJSON或TopoJSON。

## 投影

地图投影，是指按照一定的数学法则将地球椭球面上的经纬网转换到平面上，使地面的地理坐标与平面直角坐标建立起函数关系，是绘制地图的数学基础之一。由于地球是一个不可展的球体，使用物理方法将其展平会引起褶皱、拉伸和断裂，因此要使用地图投影实现由曲面向平面的转化。

下表是一些地图投影与经纬线形状特征

![image.png](https://tech-proxy.bytedance.net/tos/images/1629798282992_851dffe75825a8a90e58c2d2744982ca.png)

示例的中国地图就是采用了墨卡托投影。

## 画

经过上面的介绍，接下来只需要将二维平面内容绘制出来即可。

这里使用 d3 与 d3-geo 举例绘制上面提到的中国地图。
为了美观，使用 d3-scale-chromatic 的一个主题随意上了颜色。

```javascript
import * as d3 from 'd3';
import * as geo from 'd3-geo';
import * as d3Color from 'd3-scale-chromatic';
```

在页面中创建一个 svg 元素

```html
<svg id="svg" width="1080" height="600"></svg>
```

创建投影，设置参数

```javascript
const projection = geo
  .geoMercator() // 墨卡托投影
  .scale(450) // 投影的比例因子，可以按比例放大投影。
  .center([105, 38]); // 中心点设置为中国的中心位置 大约经度105，纬度38
```

利用得到的投影创建 path，补充颜色与边线

```javascript
const svg = d3.select('#svg');

const path = geo.geoPath(projection);
const colors = d3.scaleOrdinal(d3Color.schemeBlues[9]);

const area = svg
  .selectAll('path')
  .data(geoData.features) // 这里的geoData是下载好的中国地图GeoJSON格式数据
  .enter()
  .append('path')
  .attr('d', path)
  .attr('fill', (_, i) => {
    return colors(i);
  })
  .attr('stroke', '#fff')
  .attr('stroke-width', 1);
```

到此为止，中国地图就已经出现在了我们面前。

如果在这个基础上将北京上海两座城市标注出来呢？
定义经纬度

```javascript
const places = [
  {
    name: '北京',
    log: '116.3',
    lat: '39.9',
  },
  {
    name: '上海',
    log: '121.4',
    lat: '31.2',
  },
];
```

故技重施，用 svg 画出两个点

```javascript
const location = svg
  .selectAll('g')
  .data(places)
  .enter()
  .append('g')
  .attr('transform', (d) => {
    const coor = projection([d.log, d.lat]); // 经纬度转换为绘制坐标
    return 'translate(' + coor[0] + ',' + coor[1] + ')';
  });

// 绿色圆点
location.append('circle').attr('r', 4).attr('fill', '#a0d911');
```

现在地图上的北京与上海两地就被标注出来咯
![image.png](https://tech-proxy.bytedance.net/tos/images/1585641588757_3e99e6bb17c92be32d68dd1ab5a668a8.png)


