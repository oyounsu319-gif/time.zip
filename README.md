<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>석계역1번출구.A (11283) - 76번 실시간 도착 정보</title>
<style>
body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; padding: 24px; background-color: #f4f6f8; margin: 0; }
.container { max-width: 580px; margin: 0 auto; background: #fff; border-radius: 14px; padding: 28px; box-shadow: 0 4px 16px rgba(0,0,0,0.08); }
.header { border-bottom: 2px solid #edf2f7; padding-bottom: 16px; margin-bottom: 20px; display: flex; justify-content: space-between; align-items: center; }
.route-title-box { display: flex; align-items: center; gap: 12px; }
.badge-route { background-color: #e1effe; color: #1e429f; font-size: 24px; font-weight: 800; padding: 6px 14px; border-radius: 8px; }
h1 { font-size: 20px; margin: 0; color: #1a202c; }
.subtext { font-size: 13px; color: #718096; margin-top: 4px; }
.ars-badge { background-color: #edf2f7; color: #2d3748; padding: 2px 6px; border-radius: 4px; font-weight: bold; }

button { background: #1e429f; color: #fff; border: none; padding: 8px 16px; border-radius: 6px; cursor: pointer; font-weight: 600; font-size: 13px; }
button:hover { background: #1a365d; }

.card { background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 10px; padding: 20px; margin-bottom: 16px; }
.card-label { font-size: 13px; font-weight: 600; color: #64748b; margin-bottom: 8px; display: flex; justify-content: space-between; }

.time-main { font-size: 26px; font-weight: 800; color: #1e429f; letter-spacing: -0.5px; }
.time-urgent { color: #dc2626; }
.time-none { color: #94a3b8; font-size: 20px; }

.info-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 16px; padding-top: 14px; border-top: 1px dashed #cbd5e1; }
.info-item { font-size: 13px; color: #475569; }
.info-item b { color: #0f172a; }

.sub-card { background: #fff; border: 1px solid #e2e8f0; border-radius: 8px; padding: 14px 16px; font-size: 14px; }

.status-bar { display: flex; justify-content: space-between; align-items: center; font-size: 12px; color: #64748b; margin-top: 16px; }
.status-dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; background: #22c55e; margin-right: 6px; }
</style>
</head>
<body>
<div class="container">
  <div class="header">
    <div>
      <div class="route-title-box">
        <span class="badge-route">76</span>
        <div>
          <h1 id="station-title">석계역1번출구.A</h1>
          <div class="subtext">정류소 고유번호: <span class="ars-badge">11283</span></div>
        </div>
      </div>
    </div>
    <button onclick="handleManualRefresh()">새로고침</button>
  </div>

  <div class="card">
    <div class="card-label">
      <span>첫 번째 도착 버스</span>
      <span id="route-dir-text">차량번호: -</span>
    </div>
    <div id="first-bus-arrival" class="time-main">데이터 조회 중...</div>

    <div class="info-grid">
      <div class="info-item">남은 정류장: <b id="stops-left-text">-</b></div>
      <div class="info-item">차내 잔여좌석: <b id="congestion-text">정보 없음</b></div>
    </div>
  </div>

  <div class="sub-card">
    <div class="card-label" style="margin-bottom: 6px;">다음 버스 (배차 간격)</div>
    <div id="second-bus-arrival" style="font-weight: 600; color: #1e293b;">조회 중...</div>
  </div>

  <div class="status-bar">
    <span id="status-text"><span class="status-dot"></span>실시간 연동 대기 중</span>
    <span id="timer-text"></span>
  </div>
</div>

<script>
const SERVICE_KEY = "qx5Zwgc%2FCGU%2FXrw6ZG7DfINCRS9%2BMyU4N4GI7t0EKboj8i2HGEj0YPXue%2BHvMpi%2FEbfcb%2FPIwwWPyLOQ%2BOfizQ%3D%3D";
const STATION_ID = "110000183"; 
const TARGET_ROUTE_ID = "222000181"; 

let countdown = 15;
let timerId = null;

function handleManualRefresh() {
  loadArrivalData();
  startTimer();
}

async function requestBusData() {
  const targetUrl = `https://apis.data.go.kr/6410000/busarrivalservice/v2/getBusArrivalListv2?serviceKey=${SERVICE_KEY}&stationId=${STATION_ID}`;

  // 1차 시도: AllOrigins JSON 프록시 (origin null 완벽 호환)
  try {
    const res = await fetch(`https://api.allorigins.win/get?url=${encodeURIComponent(targetUrl)}`);
    if (res.ok) {
      const data = await res.json();
      if (data.contents) return data.contents;
    }
  } catch (e) {
    console.warn("1차 프록시 실패, 보조 프록시로 전환");
  }

  // 2차 시도: CodeTabs 백업 프록시
  const fallbackRes = await fetch(`https://api.codetabs.com/v1/proxy?quest=${encodeURIComponent(targetUrl)}`);
  if (!fallbackRes.ok) throw new Error("모든 프록시 서버 통신 실패");
  return await fallbackRes.text();
}

async function loadArrivalData() {
  const statusText = document.getElementById("status-text");
  const firstEl = document.getElementById("first-bus-arrival");
  const secondEl = document.getElementById("second-bus-arrival");
  const stopsEl = document.getElementById("stops-left-text");
  const plateEl = document.getElementById("route-dir-text");
  const congEl = document.getElementById("congestion-text");

  statusText.innerHTML = `<span class="status-dot" style="background:#eab308;"></span>데이터 갱신 중...`;

  try {
    const textData = await requestBusData();
    const parser = new DOMParser();
    const xmlDoc = parser.parseFromString(textData, "text/xml");

    // 인증키 만료/오류 확인
    const authErrMsg = xmlDoc.getElementsByTagName("returnAuthMsg")[0]?.textContent;
    const cmmErrMsg = xmlDoc.getElementsByTagName("errMsg")[0]?.textContent;
    if (authErrMsg || cmmErrMsg) {
      firstEl.innerText = "API 키 오류";
      secondEl.innerText = authErrMsg || cmmErrMsg;
      statusText.innerHTML = `<span class="status-dot" style="background:#ef4444;"></span>인증 실패`;
      return;
    }

    const resultCode = xmlDoc.getElementsByTagName("resultCode")[0]?.textContent;
    const resultMessage = xmlDoc.getElementsByTagName("resultMessage")[0]?.textContent;

    if (resultCode === "0") {
      const arrivals = xmlDoc.getElementsByTagName("busArrivalList");
      let targetBus = null;
      
      for (let i = 0; i < arrivals.length; i++) {
        if (arrivals[i].getElementsByTagName("routeId")[0]?.textContent === TARGET_ROUTE_ID) {
          targetBus = arrivals[i];
          break;
        }
      }

      if (targetBus) {
        const loc1 = targetBus.getElementsByTagName("locationNo1")[0]?.textContent;
        const time1 = targetBus.getElementsByTagName("predictTime1")[0]?.textContent;
        const plate1 = targetBus.getElementsByTagName("plateNo1")[0]?.textContent;
        const seat1 = targetBus.getElementsByTagName("remainSeatCnt1")[0]?.textContent;

        if (loc1 && Number(loc1) >= 0) {
          const numLoc1 = Number(loc1);
          const numTime1 = Number(time1);
          
          if (numLoc1 === 0 || (numLoc1 === 1 && numTime1 <= 2)) {
            firstEl.innerText = `회차지 진입/출발 대기`;
            firstEl.className = "time-main time-urgent";
            stopsEl.innerText = `출발 대기`;
          } else {
            firstEl.innerText = `약 ${time1}분 후 도착`;
            firstEl.className = (numTime1 <= 3) ? "time-main time-urgent" : "time-main";
            stopsEl.innerText = `${loc1}정거장 전`;
          }

          plateEl.innerText = `차량번호: ${plate1 || "정보없음"}`;
          congEl.innerText = (!seat1 || seat1 === "-1") ? "정보 없음" : `${seat1}석 여유`;
        } else {
          firstEl.innerText = "도착 예정 버스 없음";
          firstEl.className = "time-main time-none";
          stopsEl.innerText = "-";
          plateEl.innerText = "차량번호: -";
          congEl.innerText = "정보 없음";
        }

        const loc2 = targetBus.getElementsByTagName("locationNo2")[0]?.textContent;
        const time2 = targetBus.getElementsByTagName("predictTime2")[0]?.textContent;
        
        if (loc2 && Number(loc2) > 0) {
          secondEl.innerText = `약 ${time2}분 후 (${loc2}정거장 전)`;
        } else {
          secondEl.innerText = "다음 도착 예정 차량 없음";
        }
      } else {
        firstEl.innerText = "운행 중인 차량 없음";
        firstEl.className = "time-main time-none";
        secondEl.innerText = "-";
        stopsEl.innerText = "-";
        plateEl.innerText = "차량번호: -";
        congEl.innerText = "정보 없음";
      }
      
      statusText.innerHTML = `<span class="status-dot"></span>정상 연동 완료 (${new Date().toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })})`;
      
    } else if (resultCode === "4") {
      firstEl.innerText = "운행 중인 차량 없음";
      firstEl.className = "time-main time-none";
      secondEl.innerText = "-";
      stopsEl.innerText = "-";
      plateEl.innerText = "차량번호: -";
      congEl.innerText = "정보 없음";
      statusText.innerHTML = `<span class="status-dot"></span>정상 연동 완료 (${new Date().toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit', second: '2-digit' })})`;
    } else {
      firstEl.innerText = "API 응답 오류";
      secondEl.innerText = resultMessage || "알 수 없는 오류";
      statusText.innerHTML = `<span class="status-dot" style="background:#ef4444;"></span>오류 (${resultCode})`;
    }
  } catch (err) {
    console.error(err);
    firstEl.innerText = "통신 실패";
    secondEl.innerText = "프록시/네트워크 차단";
    statusText.innerHTML = `<span class="status-dot" style="background:#ef4444;"></span>네트워크 에러`;
  }
}

function startTimer() {
  countdown = 15;
  if (timerId) clearInterval(timerId);

  document.getElementById("timer-text").innerText = `${countdown}초 후 자동 갱신`;
  timerId = setInterval(() => {
    countdown--;
    document.getElementById("timer-text").innerText = `${countdown}초 후 자동 갱신`;
    if (countdown <= 0) {
      loadArrivalData();
      countdown = 15;
    }
  }, 1000);
}

loadArrivalData();
startTimer();
</script>
</body>
</html>
