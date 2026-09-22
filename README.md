<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>석계역 버스 대기시간 활용 맵 (카카오맵 & 실시간 API)</title>
  
  <link rel="stylesheet" as="style" crossorigin href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css" />
  
  <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=5db3c0d2f986fe340feb49ca0f118e84"></script>

  <style>
    :root {
      --bg-color: #F4F6F8;
      --card-bg: #FFFFFF;
      --primary-blue: #1E429F;
      --text-main: #1A202C;
      --text-sub: #4A5568;
      --border-color: #E2E8F0;
      
      --pin-green: #22C55E;
      --pin-yellow: #EAB308;
      --pin-orange: #F97316;
    }
    
    * { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Pretendard', sans-serif; }
    body, html { width: 100%; height: 100%; overflow: hidden; background-color: var(--bg-color); }
    
    /* 지도 영역 */
    #map { position: absolute; top: 0; bottom: 0; width: 100%; z-index: 1; }
    
    /* 상단 헤더 */
    .header {
      position: absolute; top: 0; left: 0; right: 0; z-index: 10;
      background: rgba(255, 255, 255, 0.95); backdrop-filter: blur(8px);
      padding: 16px 20px; border-bottom: 1px solid var(--border-color);
      box-shadow: 0 2px 10px rgba(0,0,0,0.05); display: flex; flex-direction: column; gap: 10px;
    }
    .header-top { display: flex; justify-content: space-between; align-items: center; }
    .station-info { font-size: 18px; font-weight: 700; color: var(--text-main); }
    .bus-badge { background: var(--primary-blue); color: white; padding: 4px 10px; border-radius: 6px; font-size: 14px; font-weight: 600; }
    .time-board { display: flex; justify-content: space-between; align-items: center; background: var(--bg-color); padding: 12px; border-radius: 8px; }
    .time-left { font-size: 24px; font-weight: 800; color: var(--primary-blue); }
    .realtime-text { font-size: 12px; color: var(--text-sub); display: block; margin-top: 4px; font-weight: 600; }
    
    /* 필터 토글 */
    .filter-wrap { display: flex; gap: 8px; margin-top: 4px; }
    .filter-btn { background: white; border: 1px solid var(--border-color); padding: 6px 12px; border-radius: 20px; font-size: 12px; color: var(--text-sub); cursor: pointer; transition: 0.2s; }
    .filter-btn.active { background: var(--primary-blue); color: white; border-color: var(--primary-blue); }

    /* 플로팅 버튼 */
    .fabs { position: absolute; right: 16px; bottom: 260px; z-index: 10; display: flex; flex-direction: column; gap: 10px; }
    .fab { width: 44px; height: 44px; background: white; border-radius: 50%; display: flex; justify-content: center; align-items: center; box-shadow: 0 4px 12px rgba(0,0,0,0.15); border: 1px solid var(--border-color); cursor: pointer; font-size: 18px; }
    
    /* 바텀 시트 (장소 리스트) */
    .bottom-sheet {
      position: absolute; bottom: 0; left: 0; right: 0; z-index: 20;
      background: var(--card-bg); height: 240px; border-radius: 20px 20px 0 0;
      box-shadow: 0 -4px 20px rgba(0,0,0,0.1); padding: 20px;
      display: flex; flex-direction: column; gap: 12px; overflow-y: auto;
    }
    .sheet-handle { width: 40px; height: 4px; background: #CBD5E1; border-radius: 2px; margin: 0 auto 10px; }
    
    /* 추천 장소 카드 */
    .place-card {
      border: 1px solid var(--border-color); border-radius: 12px; padding: 14px;
      display: flex; justify-content: space-between; align-items: center; cursor: pointer; transition: 0.2s;
    }
    .place-card:active { background: var(--bg-color); }
    .place-info h3 { font-size: 16px; color: var(--text-main); margin-bottom: 4px; display: flex; align-items: center; gap: 6px; }
    .place-info p { font-size: 13px; color: var(--text-sub); }
    .walk-time { font-size: 14px; font-weight: 700; color: var(--primary-blue); }
    
    /* 커스텀 마커 */
    .custom-marker { width: 16px; height: 16px; border-radius: 50%; border: 2px solid white; box-shadow: 0 2px 4px rgba(0,0,0,0.3); cursor: pointer; }
    .marker-green { background: var(--pin-green); }
    .marker-yellow { background: var(--pin-yellow); }
    .marker-orange { background: var(--pin-orange); }
    .marker-station { width: 24px; height: 24px; background: var(--primary-blue); border-radius: 50%; border: 3px solid white; cursor: default; }

    /* 대기 모드 UI (오버레이) */
    .wait-mode-overlay {
      display: none; position: absolute; top: 145px; left: 20px; right: 20px; z-index: 30;
      background: white; border: 2px solid var(--primary-blue); border-radius: 12px; padding: 20px;
      box-shadow: 0 8px 24px rgba(30,66,159,0.2); text-align: center;
    }
    .wait-mode-overlay h2 { font-size: 16px; color: var(--text-sub); margin-bottom: 8px; }
    .stay-time { font-size: 32px; font-weight: 800; color: var(--primary-blue); margin-bottom: 16px; }
    .btn-group { display: flex; gap: 10px; }
    .btn-action { flex: 1; padding: 12px; border-radius: 8px; font-weight: 600; font-size: 14px; cursor: pointer; border: none; }
    .btn-next-bus { background: var(--bg-color); color: var(--text-main); }
    .btn-wait-here { background: var(--primary-blue); color: white; }

    /* 시각적 알람 (시간 임박 시) */
    .flash-warning { animation: flash 1s infinite; }
    @keyframes flash { 0% { background-color: rgba(239, 68, 68, 0.1); } 50% { background-color: rgba(239, 68, 68, 0.4); } 100% { background-color: rgba(239, 68, 68, 0.1); } }
  </style>
</head>
<body>

  <header class="header">
    <div class="header-top">
      <div class="station-info">석계역 1번 출구 (B구역)</div>
      <div class="bus-badge">1132번</div>
    </div>
    <div class="time-board">
      <div>
        <div style="font-size: 12px; color: var(--text-sub);">실시간 버스 도착까지</div>
        <div class="time-left" id="bus-timer">조회 중...</div>
        <span class="realtime-text" id="bus-raw-msg">API 연결 중...</span>
      </div>
      <button class="filter-btn" onclick="addNextBus()">+ 다음 버스 타기</button>
    </div>
    <div class="filter-wrap">
      <button class="filter-btn active">전체</button>
      <button class="filter-btn">♿ 무단차</button>
      <button class="filter-btn">🛗 엘리베이터</button>
    </div>
  </header>

  <div class="wait-mode-overlay" id="wait-overlay">
    <h2>지금 <span id="target-place-name">카페</span>에서 머물 수 있는 시간</h2>
    <div class="stay-time" id="stay-timer">00분 00초</div>
    <div class="btn-group">
      <button class="btn-action btn-next-bus" onclick="cancelWait()">취소</button>
      <button class="btn-action btn-wait-here" onclick="registerWaitToServer()">이곳에서 대기 (서버 연동)</button>
    </div>
  </div>

  <div id="map"></div>

  <div class="fabs">
    <div class="fab" onclick="moveToStation()">🎯</div>
    <div class="fab" onclick="alert('내 위치 추적 기능 (GPS 필요)')">🧭</div>
  </div>

  <main class="bottom-sheet" id="place-list">
    <div class="sheet-handle"></div>
  </main>

<script>
// --- [1. API KEY 및 기본 설정] ---
const KAKAO_KEY = "5db3c0d2f986fe340feb49ca0f118e84"; 
const BUS_API_KEY = "qx5Zwgc%2FCGU%2FXrw6ZG7DfINCRS9%2BMyU4N4GI7t0EKboj8i2HGEj0YPXue%2BHvMpi%2FEbfcb%2FPIwwWPyLOQ%2BOfizQ%3D%3D";
const ARS_ID = "11283"; // 석계역 1번출구.A
const TARGET_BUS = "1132"; // 목표 버스

let busTimeSeconds = 0; // 초 단위 남은 시간
let rawBusMessage = "데이터 조회 중...";
let currentSelectedPlace = null;
let isNextBusMode = false;

// 카카오맵 좌표 기준 (위도, 경도)
const stationLatLng = new kakao.maps.LatLng(37.6150, 127.0652); 

const places = [
  { id: 1, name: '메가커피 석계역점', type: '카페', color: 'green', walkMin: 2, lat: 37.6152, lng: 127.0655, desc: '테이크아웃 추천 (도보 2분)' },
  { id: 2, name: '소품샵 가챠폰', type: '쇼핑', color: 'yellow', walkMin: 5, lat: 37.6155, lng: 127.0660, desc: '가벼운 구경거리 (도보 5분)' },
  { id: 3, name: '올리브영 석계점', type: '쇼핑', color: 'yellow', walkMin: 6, lat: 37.6148, lng: 127.0662, desc: '코스메틱 쇼핑 (도보 6분)' },
  { id: 4, name: '역전우동 0410', type: '식당', color: 'orange', walkMin: 10, lat: 37.6160, lng: 127.0670, desc: '간단한 식사 (도보 10분)' }
];

// --- [2. 카카오맵 초기화] ---
const mapContainer = document.getElementById('map');
const mapOption = {
    center: stationLatLng,
    level: 3 // 확대 레벨 (작을수록 확대됨)
};
const map = new kakao.maps.Map(mapContainer, mapOption);

// 정류장 마커 표시 (커스텀 오버레이)
new kakao.maps.CustomOverlay({
    position: stationLatLng,
    content: '<div class="custom-marker marker-station"></div>',
    map: map
});

// 장소 마커 등록
places.forEach(place => {
    const content = document.createElement('div');
    content.className = `custom-marker marker-${place.color}`;
    content.onclick = () => selectPlace(place);
    
    new kakao.maps.CustomOverlay({
        position: new kakao.maps.LatLng(place.lat, place.lng),
        content: content,
        clickable: true,
        map: map
    });
});

let routePolyline = null; // 경로 선

// --- [3. 실시간 버스 API 호출 로직] ---
async function fetchRealtimeBus() {
  if(isNextBusMode) return; // '다음 버스 타기' 모드일 때는 실시간 업데이트 일시정지
  
  // 브라우저 캐시 방지를 위해 현재 시간 추가 (데이터 갱신용)
  const url = `http://ws.bus.go.kr/api/rest/stationinfo/getStationByUid?serviceKey=${BUS_API_KEY}&arsId=${ARS_ID}&resultType=json&_t=${Date.now()}`;
  const proxy = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`;
  
  try {
    const res = await fetch(proxy);
    const raw = await res.json();
    const data = JSON.parse(raw.contents);
    
    if (data.msgHeader && data.msgHeader.headerCd === "0") {
      const items = data.msgBody?.itemList || [];
      const bus = items.find(b => b.rtNm === TARGET_BUS);
      
      if (bus) {
        rawBusMessage = bus.arrmsg1; // 예: "5분후[2번째 전]"
        
        // 정규식으로 API 문자열에서 '분' 또는 '초' 추출
        const minMatch = rawBusMessage.match(/(\d+)분/);
        const secMatch = rawBusMessage.match(/(\d+)초/);
        
        if (minMatch || secMatch) {
            let totalSec = 0;
            if(minMatch) totalSec += parseInt(minMatch[1]) * 60;
            if(secMatch) totalSec += parseInt(secMatch[1]);
            busTimeSeconds = totalSec;
        } else if (rawBusMessage.includes("곧 도착") || rawBusMessage.includes("도착 임박")) {
            busTimeSeconds = 60; // 곧 도착은 1분으로 설정
        } else if (rawBusMessage.includes("종료") || rawBusMessage.includes("없음")) {
            busTimeSeconds = 0;
        }
      } else {
        rawBusMessage = "해당 노선 정보 없음";
      }
    }
  } catch(e) {
    console.error("버스 데이터 불러오기 실패", e);
    rawBusMessage = "API 통신 에러";
  }
}

// --- [4. UI 및 상호작용 로직] ---
function renderCards() {
  const listEl = document.getElementById('place-list');
  listEl.innerHTML = '<div class="sheet-handle"></div>';
  
  places.forEach(place => {
    const card = document.createElement('div');
    card.className = 'place-card';
    card.onclick = () => selectPlace(place);
    card.innerHTML = `
      <div class="place-info">
        <h3><span style="display:inline-block; width:10px; height:10px; border-radius:50%; background:var(--pin-${place.color})"></span> ${place.name}</h3>
        <p>${place.desc}</p>
      </div>
      <div class="walk-time">🚶 ${place.walkMin}분</div>
    `;
    listEl.appendChild(card);
  });
}

function selectPlace(place) {
  currentSelectedPlace = place;
  
  // 지도 부드럽게 이동
  map.panTo(new kakao.maps.LatLng(place.lat, place.lng));
  
  // 역산 타이머 오버레이 표시
  document.getElementById('target-place-name').innerText = place.name;
  document.getElementById('wait-overlay').style.display = 'block';
  
  // 도보 점선 그리기 (카카오맵 Polyline)
  if (routePolyline) routePolyline.setMap(null);
  routePolyline = new kakao.maps.Polyline({
      path: [
          new kakao.maps.LatLng(place.lat, place.lng),
          stationLatLng
      ],
      strokeWeight: 4, strokeColor: '#1E429F', strokeOpacity: 0.8, strokeStyle: 'dashed'
  });
  routePolyline.setMap(map);
}

function moveToStation() {
  map.panTo(stationLatLng);
}

function addNextBus() {
  isNextBusMode = true; // 실시간 연동 중단
  busTimeSeconds += 20 * 60; // 20분 강제 추가
  rawBusMessage = "다음 버스로 연장됨 (실시간 중단)";
  alert('목표 버스가 다음 버스로 변경되었습니다. (+20분)\n(이후 카운트다운은 자체 타이머로 돌아갑니다.)');
  updateTimerUI();
}

function cancelWait() {
  currentSelectedPlace = null;
  document.getElementById('wait-overlay').style.display = 'none';
  if(routePolyline) { routePolyline.setMap(null); routePolyline = null; }
}

function updateTimerUI() {
  const m = Math.floor(busTimeSeconds / 60);
  const s = busTimeSeconds % 60;
  
  if (busTimeSeconds > 0) {
      document.getElementById('bus-timer').innerText = `${m}분 ${s.toString().padStart(2,'0')}초`;
  } else {
      document.getElementById('bus-timer').innerText = "도착 임박/정보 없음";
  }
  document.getElementById('bus-raw-msg').innerText = `서울시 API: ${rawBusMessage}`;

  // 체류 시간 역산 계산
  if (currentSelectedPlace) {
    const stayTimeSec = busTimeSeconds - (currentSelectedPlace.walkMin * 60);
    const stayTimerEl = document.getElementById('stay-timer');
    
    if (stayTimeSec > 0) {
      const sm = Math.floor(stayTimeSec / 60);
      const ss = stayTimeSec % 60;
      stayTimerEl.innerText = `${sm}분 ${ss.toString().padStart(2,'0')}초`;
      stayTimerEl.style.color = 'var(--primary-blue)';
      document.body.classList.remove('flash-warning');
    } else {
      stayTimerEl.innerText = "지금 당장 출발하세요!";
      stayTimerEl.style.color = '#EF4444';
      document.body.classList.add('flash-warning');
    }
  }
}

// 백엔드 연동 (FastAPI)
async function registerWaitToServer() {
  const backendUrl = 'https://YOUR_REPLIT_URL/register'; // 추후 Replit 주소로 변경 필요
  
  const payload = {
    station: "석계역 1번 출구", target_bus: TARGET_BUS,
    stay_location: currentSelectedPlace.name,
    user_location: { lat: currentSelectedPlace.lat, lng: currentSelectedPlace.lng }
  };

  try {
    const response = await fetch(backendUrl, {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
    const result = await response.json();
    alert(`서버 응답: ${result.message}\n(이제 백그라운드에서도 알람이 유지됩니다.)`);
  } catch (error) {
    alert('백엔드 서버 연동은 코드에 백엔드 주소를 직접 입력해야 작동합니다.');
  }
}

// ------------------------------
// 프로그램 실행 흐름
// ------------------------------
renderCards();
fetchRealtimeBus(); 

// 1초마다 화면 타이머 갱신
setInterval(() => {
  if (busTimeSeconds > 0) busTimeSeconds--;
  updateTimerUI();
}, 1000);

// 30초마다 실제 버스 API 최신화 (시간 오차 보정용)
setInterval(() => {
  fetchRealtimeBus();
}, 30000);

</script>
</body>
</html>
