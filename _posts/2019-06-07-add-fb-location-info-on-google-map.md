---
title: "用FB打卡紀錄在Google Map打點"
layout: post
tags: [Google Map, Facebook, Web]
published: true
comments: true
---
這一篇我們要用上一篇取得的資料來打點了!

## Google MAP API Overview

先說個不幸的消息, Google MAP API從2018年七月開始需要收費, 引述[Google地圖改變收費模式，API計費變嚴變貴影響政府機構](https://www.bnext.com.tw/article/49903/google-maps-api-charge)的說明:

> 今年5月2日Google發文預告，將在7月16日開發，調整收費模式。每位用到Google地圖架設服務的開發者，不論是否達到Google地圖收費的標準，都得申請API key，並且綁訂信用卡卡號。而每位都有每位200美元的免費額度，超過才會需要繳錢給Google。

不過我們不用付費還是可以像這篇一樣用最基本的功能來Demo, 只是會跳出警示跟浮水印, 如果真的要當產品供人大量使用的就要三思了! 

有顧慮的話可以考慮開源免費的 [OpenStreet Map](https://www.openstreetmap.org/) , 圖資主要以國外的較充足且API可能相較起來沒這麼強大, 畢竟一分錢一分貨嘛!

Google Map提供了 **Maps Javascript API**, **iOS/ Android SDK**等服務, 主要功能就是讓使用者能夠在地圖上顯示自訂的地點/路徑等資訊, 這篇講的主要為網頁版的 **Maps Javascript API**.

### Hello Taipei 101!
我稍微改了一下[官網](https://developers.google.com/maps/documentation/javascript/tutorial?hl=zh-tw)提供的 `Hello World` 範例, 我把中心點改到台北101, 然後在101打一個點(Marker). 

語法非常簡單, 在load Google Map JS成功之後會呼叫 `callback initMap()`.

`initMap()`首先在`<div id="map"></div>`建立一個**Map Object**, `center` 就是地圖的中心.<br>
接著, 新增 **Marker Object**在map上. Marker可以指定 **click listener**, 點擊後跳出自訂html內容的 **infoWindow**.

<iframe width="100%" height="300" src="//jsfiddle.net/t6847kimo/9h7fz1aw/6/embedded/" allowfullscreen="allowfullscreen" allowpaymentrequest frameborder="0"></iframe>


## Full Code
因為Google Map API實在太簡單(讓人都變笨了XD), 上面的範例已經快做完了, 接下來只要把上次撈出來的資料打點出來就大功告成!

比較需要注意的是FB data的格式稍微複雜一些, 建議可以先用上次提到的[圖型API測試工具](https://developers.facebook.com/tools/explorer/?method=GET&path=me?fields=id,name,posts.limit%2850%29%7Bplace,message%7D&version=v3.3)或是console output出來看一下內容.

一樣記得替換掉以下`YOUR_APP_ID`  &  `YOUR_ACCESS_TOKEN` & `YOUR_API_KEY`, 如果跑在local端則可以把`key=YOUR_API_KEY&`這段拿掉純demo用.

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Google Map with FB posts location info</title>
    <meta name="viewport" content="initial-scale=1.0">
    <meta charset="utf-8">
    <style>
      /* Always set the map height explicitly to define the size of the div
       * element that contains the map. */
      #map {
        height: 100%;
      }
      /* Optional: Makes the sample page fill the window. */
      html, body {
        height: 100%;
        margin: 0;
        padding: 0;
      }
    </style>
  </head>
  <body>
    <div id="map"></div>
    <!-- Replace the value of the key parameter with your own API key. -->
    <script src="https://maps.googleapis.com/maps/api/js?key=YOUR_API_KEY&callback=initMap" async defer></script>
    <script src="https://connect.facebook.net/en_US/sdk.js"></script>
    <script>
      function initFbData(map){
        FB.init({
          appId            : 'YOUR_APP_ID',
          autoLogAppEvents : true,
          xfbml            : true,
          version          : 'v3.2'
        });
        FB.api(
          '/me',
          'GET',
          {
            "fields":"id,name,posts.limit(50){place,message}",
            "access_token": "YOUR_ACCESS_TOKEN"
          },
          function(response) {
            // uncomment to show response content
            // console.log(response);
            response.posts.data.forEach(function(element){
              // filter posts which without place
              if(element.place == null) return;
              var post_place = element.place;
              // add marker
              var pos = {lat: post_place.location.latitude, lng: post_place.location.longitude};
              var marker = new google.maps.Marker({
                position: pos,
                map: map,
                title: post_place.name
              });
              // add infoWindow
              var infoWindow = createInfoWindow(post_place.name, element.message);
              marker.addListener('click', 
                function() {
                  infoWindow.open(map, marker);
                }
              );
            });
          }
        );
      }

      function createInfoWindow(title, content){
        // generate html of infoWindow
        var contentString = '<div class="myInfoWindow">'+
          '<h1 class="myFirstHeading">'+ title + '</h1>'+
          '<div class="myInfoWindowContent">'+ content + '</div>'+
          '</div>';
          
        var infowindow = new google.maps.InfoWindow({
          content: contentString
        });
        return infowindow;
      }

      function initMap() {
        // Create Map object, center is Taipei 101
        var center = {lat: 25.0339639, lng: 121.5622835};
        var map = new google.maps.Map(document.getElementById('map'), {
          zoom: 8,
          center: center
        });
        initFbData(map);
      }
    </script>
  </body>
</html>
```
![Google Map with FB posts location info](https://raw.githubusercontent.com/t6847kimo/blog/master/assets/img/Google%20Map%20Demo.png)

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTc5MjYzNDI3MCwtMTM2NjQzODk1NCwtMT
I2OTY1NDU0MywyMTI2Nzg1MjM1LC0yMTE5MjUzOTM2LC0zMTQz
ODYwMiw0NDYzNzczOSwtNTk4NjMwNzEyLDE5MjY2NjU1NjddfQ
==
-->