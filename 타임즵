// --- [3. 실시간 버스 API 호출 로직] ---
async function fetchRealtimeBus() {
  if(isNextBusMode) return; 
  
  const url = `http://ws.bus.go.kr/api/rest/stationinfo/getStationByUid?serviceKey=${BUS_API_KEY}&arsId=${ARS_ID}&resultType=json&_t=${Date.now()}`;
  const proxy = `https://api.allorigins.win/get?url=${encodeURIComponent(url)}`;
  
  try {
    const res = await fetch(proxy);
    const raw = await res.json();
    
    // [추가된 부분] 공공데이터포털 에러(XML) 발생 시 예외 처리
    if (raw.contents.trim().startsWith("<")) {
        console.error("API 서버 에러(XML 반환): 인증키 오류 또는 트래픽 초과");
        rawBusMessage = "API 인증 오류 (키 확인 필요)";
        busTimeSeconds = 0;
        updateTimerUI();
        return;
    }

    const data = JSON.parse(raw.contents);
    
    if (data.msgHeader && data.msgHeader.headerCd === "0") {
      const items = data.msgBody?.itemList || [];
      const bus = items.find(b => b.rtNm === TARGET_BUS);
      
      if (bus) {
        rawBusMessage = bus.arrmsg1; 
        
        const minMatch = rawBusMessage.match(/(\d+)분/);
        const secMatch = rawBusMessage.match(/(\d+)초/);
        
        let totalSec = 0;
        if (minMatch || secMatch) {
            if(minMatch) totalSec += parseInt(minMatch[1]) * 60;
            if(secMatch) totalSec += parseInt(secMatch[1]);
            busTimeSeconds = totalSec;
        } else if (rawBusMessage.includes("곧 도착") || rawBusMessage.includes("임박")) {
            busTimeSeconds = 60; 
        } else if (rawBusMessage.includes("종료") || rawBusMessage.includes("없음")) {
            busTimeSeconds = 0;
        }
      } else {
        rawBusMessage = "1132번 운행 정보 없음";
        busTimeSeconds = 0;
      }
    } else {
        rawBusMessage = data.msgHeader?.headerMsg || "API 응답 오류";
    }
  } catch(e) {
    console.error("버스 데이터 불러오기 실패", e);
    rawBusMessage = "네트워크 통신 에러";
  }
  updateTimerUI(); // UI 즉시 반영
}
