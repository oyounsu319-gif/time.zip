<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8"/>
  <title>카카오 지도 확인</title>
</head>
<body style="margin: 0; padding: 20px;">

  <!-- 1. 높이가 지정된 지도 영역 -->
  <div id="map" style="width: 100%; max-width: 600px; height: 400px; border: 1px solid #ccc;"></div>

  <!-- 2. 실제 키가 적용된 카카오 스크립트 -->
  <script type="text/javascript" src="https://dapi.kakao.com/v2/maps/sdk.js?appkey=5db3c0d2f986fe340feb49ca0f118e84"></script>

  <!-- 3. 지도 생성 스크립트 -->
  <script>
    var container = document.getElementById('map');
    var options = {
      center: new kakao.maps.LatLng(37.61528, 127.06528), // 석계역 1번 출구 좌표
      level: 3
    };
    var map = new kakao.maps.Map(container, options);

    // 마커 표시
    var marker = new kakao.maps.Marker({
      position: options.center
    });
    marker.setMap(map);
  </script>
</body>
</html>
