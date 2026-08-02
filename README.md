/**
 * GOOGLEFINANCE 데이터 허브 (임의 종목 온디맨드)
 * ═══════════════════════════════════════════════════════════════
 * 백테스터 앱이 종목을 요청하면 GOOGLEFINANCE로 이력을 받아
 * 시트에 누적 보관한 뒤 압축 JSON으로 돌려줍니다.
 *
 * 설치
 *  1. 구글시트 → 확장 프로그램 → Apps Script → 이 코드를 붙여넣고 저장
 *  2. setup 을 한 번 실행해 권한 승인
 *  3. 배포 → 새 배포 → 유형: 웹 앱
 *       실행: 나 /  액세스 권한: 모든 사용자
 *     발급된 /exec 주소를 백테스터 앱의 "구글시트 웹앱 URL" 칸에 붙여넣기
 *  4. (선택) 트리거 → refreshWatchlist → 시간 기반 → 매일 오전 8시
 *
 * 앱에서 종목을 추가하면 이 시트의 관심목록에 자동 등록되고,
 * 이후 트리거가 매일 갱신하므로 앱을 열지 않아도 데이터가 최신으로 유지됩니다.
 */

var TOKEN_PROP = 'ACCESS_TOKEN';
var CONFIG_SHEET = '_CONFIG';
var ARCH_PREFIX = '_ARCH_';
var WATCH_SHEET = '_WATCHLIST';
var SCRATCH = '_GF_SCRATCH';
var DEFAULT_YEARS = 10;
var WAIT_MS = 1200;
var WAIT_TRIES = 30;

/* ═══════════════ 티커 해석 ═══════════════ */

/** 'SOXL' 처럼 거래소 접두사가 없으면 후보를 순서대로 시도한다 */
function tickerCandidates_(raw) {
  var t = String(raw || '').trim().toUpperCase();
  if (!t) return [];
  if (t.indexOf(':') >= 0) return [t];
  return [t, 'NASDAQ:' + t, 'NYSEARCA:' + t, 'NYSE:' + t, 'INDEXCBOE:' + t, 'INDEXSP:' + t];
}

/* ═══════════════ 수집 ═══════════════ */

/** 티커가 http로 시작하면 CSV 수집, 아니면 GOOGLEFINANCE */
function pullSeries_(ss, rawTicker, years) {
  if (/^https?:\/\//i.test(String(rawTicker || '').trim())) {
    return { ticker: String(rawTicker).trim(), rows: fetchCsv_(String(rawTicker).trim()) };
  }
  return pullGoogleFinance_(ss, rawTicker, years);
}

/** 날짜 문자열을 yyyy-MM-dd로. MM/DD/YYYY, YYYY-MM-DD, DD-MMM-YY 지원 */
function normDate_(raw) {
  var s = String(raw || '').trim();
  var m;
  if ((m = s.match(/^(\d{4})-(\d{1,2})-(\d{1,2})/))) {
    return m[1] + '-' + pad2_(m[2]) + '-' + pad2_(m[3]);
  }
  if ((m = s.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})/))) {
    return m[3] + '-' + pad2_(m[1]) + '-' + pad2_(m[2]);
  }
  var d = new Date(s);
  if (!isNaN(d.getTime()) && d.getFullYear() > 1980 && d.getFullYear() < 2100) {
    return Utilities.formatDate(d, 'UTC', 'yyyy-MM-dd');
  }
  return null;
}
function pad2_(x) { x = String(x); return x.length < 2 ? '0' + x : x; }

/**
 * 외부 CSV를 서버에서 받아 [['yyyy-MM-dd', close], ...] 로 변환한다.
 * 브라우저가 아니라 구글 서버가 요청하므로 CORS 제한이 없다.
 * 헤더에서 날짜/종가 열을 찾고, 없으면 1번째·마지막 숫자열을 쓴다.
 */
function fetchCsv_(url) {
  var res = UrlFetchApp.fetch(url, { muteHttpExceptions: true, followRedirects: true });
  var code = res.getResponseCode();
  if (code !== 200) throw new Error('CSV 요청 실패 (HTTP ' + code + ')');

  var rows = Utilities.parseCsv(res.getContentText());
  if (!rows || rows.length < 2) throw new Error('CSV가 비어 있습니다');

  var di = -1, ci = -1, start = 0;
  for (var r = 0; r < Math.min(rows.length, 10); r++) {
    var head = rows[r].map(function (x) { return String(x).trim().toLowerCase(); });
    var fd = -1, fc = -1;
    for (var k = 0; k < head.length; k++) {
      if (fd < 0 && /^(date|날짜|일자|datetime|time)$/.test(head[k])) fd = k;
      if (/^(close|adj close|종가|value|price|last)$/.test(head[k])) fc = k;
    }
    if (fd >= 0 && fc >= 0) { di = fd; ci = fc; start = r + 1; break; }
  }
  if (di < 0) {
    for (var r2 = 0; r2 < Math.min(rows.length, 20); r2++) {
      if (normDate_(rows[r2][0]) && rows[r2].length >= 2) { di = 0; ci = rows[r2].length - 1; start = r2; break; }
    }
  }
  if (di < 0) throw new Error('날짜·종가 열을 찾지 못했습니다');

  var out = [];
  for (var i = start; i < rows.length; i++) {
    if (!rows[i] || rows[i].length <= Math.max(di, ci)) continue;
    var d = normDate_(rows[i][di]);
    var v = parseFloat(String(rows[i][ci]).replace(/[$,\s"]/g, ''));
    if (d && isFinite(v) && v > 0) out.push([d, v]);
  }
  if (out.length < 30) throw new Error('유효한 행이 너무 적습니다 (' + out.length + ')');
  return out;
}

/**
 * @return {{ticker:string, rows:Array.<Array>}} rows = [['yyyy-MM-dd', close], ...]
 */
function pullGoogleFinance_(ss, rawTicker, years) {
  var cands = tickerCandidates_(rawTicker);
  if (!cands.length) throw new Error('티커가 비어 있습니다');

  var sh = ss.getSheetByName(SCRATCH) || ss.insertSheet(SCRATCH);
  sh.hideSheet();
  var lastErr = '알 수 없는 오류';

  for (var ci = 0; ci < cands.length; ci++) {
    var tk = cands[ci];
    try {
      sh.clear();
      var f = '=IFERROR(GOOGLEFINANCE("' + tk + '","close",'
        + 'WORKDAY(TODAY(),-' + Math.round((years || DEFAULT_YEARS) * 261) + '),'
        + 'TODAY(),"DAILY"),"__ERR__")';
      sh.getRange(1, 1).setFormula(f);
      SpreadsheetApp.flush();

      var vals = null, ok = false;
      for (var i = 0; i < WAIT_TRIES; i++) {
        Utilities.sleep(WAIT_MS);
        SpreadsheetApp.flush();
        vals = sh.getDataRange().getValues();
        var a = vals[0][0];
        if (a === '__ERR__') { lastErr = tk + ' 지원하지 않음'; break; }
        if (a === '' || a === 'Loading...' || vals.length < 2) continue;
        ok = true; break;
      }
      if (!ok) { if (lastErr.indexOf('지원') < 0) lastErr = tk + ' 응답 없음'; continue; }

      var tz = ss.getSpreadsheetTimeZone(), out = [];
      for (var r = 0; r < vals.length; r++) {
        var d = vals[r][0], c = vals[r].length > 1 ? vals[r][1] : null;
        if (!(d instanceof Date)) continue;
        var v = parseFloat(c);
        if (!isFinite(v) || v <= 0) continue;
        out.push([Utilities.formatDate(d, tz, 'yyyy-MM-dd'), v]);
      }
      if (out.length < 30) { lastErr = tk + ' 데이터가 너무 적음(' + out.length + '행)'; continue; }
      return { ticker: tk, rows: out };
    } catch (e) {
      lastErr = tk + ' — ' + e.message;
    }
  }
  throw new Error(lastErr);
}

/** 아카이브에 누적 병합 (같은 날짜는 갱신, 기존 이력은 삭제하지 않음) */
function mergeArchive_(ss, id, rows) {
  var sh = ss.getSheetByName(ARCH_PREFIX + id) || ss.insertSheet(ARCH_PREFIX + id);
  var tz = ss.getSpreadsheetTimeZone(), map = {};

  if (sh.getLastRow() > 1) {
    var old = sh.getRange(2, 1, sh.getLastRow() - 1, 2).getValues();
    for (var i = 0; i < old.length; i++) {
      var d = old[i][0], v = parseFloat(old[i][1]);
      if (d instanceof Date) d = Utilities.formatDate(d, tz, 'yyyy-MM-dd');
      d = String(d).slice(0, 10);
      if (/^\d{4}-\d{2}-\d{2}$/.test(d) && isFinite(v) && v > 0) map[d] = v;
    }
  }
  for (var j = 0; j < rows.length; j++) map[rows[j][0]] = rows[j][1];

  var keys = Object.keys(map).sort();
  var body = keys.map(function (k) { return [k, map[k]]; });

  sh.clear();
  sh.getRange(1, 1, 1, 2).setValues([['날짜', '종가']]).setFontWeight('bold');
  if (body.length) {
    sh.getRange(2, 1, body.length, 2).setValues(body);
    sh.getRange(2, 1, body.length, 1).setNumberFormat('@');
  }
  sh.setFrozenRows(1);
  sh.hideSheet();
  return body.length;
}

/* ═══════════════ 관심목록 ═══════════════ */

function watchSheet_(ss) {
  var sh = ss.getSheetByName(WATCH_SHEET);
  if (!sh) {
    sh = ss.insertSheet(WATCH_SHEET);
    sh.getRange(1, 1, 1, 4).setValues([['ID', '티커', '최종갱신', '행수']]).setFontWeight('bold');
    sh.setFrozenRows(1);
  }
  return sh;
}
function watchList_(ss) {
  var sh = watchSheet_(ss);
  if (sh.getLastRow() < 2) return [];
  return sh.getRange(2, 1, sh.getLastRow() - 1, 2).getValues()
    .filter(function (r) { return r[0]; })
    .map(function (r) { return { id: String(r[0]).trim(), ticker: String(r[1]).trim() }; });
}
function watchUpsert_(ss, id, ticker, rowCount) {
  var sh = watchSheet_(ss);
  var list = watchList_(ss);
  var now = Utilities.formatDate(new Date(), ss.getSpreadsheetTimeZone(), 'yyyy-MM-dd HH:mm');
  for (var i = 0; i < list.length; i++) {
    if (list[i].id === id) {
      sh.getRange(i + 2, 2, 1, 3).setValues([[ticker, now, rowCount]]);
      return;
    }
  }
  sh.appendRow([id, ticker, now, rowCount]);
}

/* ═══════════════ 공개 함수 ═══════════════ */

function setup() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  watchSheet_(ss);
  ss.toast('준비되었습니다. generateToken 실행 후 배포하세요.', '설정 완료', 10);
}

/* ═══════════════ 접근 토큰 ═══════════════ */

/**
 * 접근 토큰을 새로 만들어 저장하고 화면에 보여준다.
 * 이 값을 백테스터 앱의 "접근 토큰" 칸에 붙여넣으세요.
 * 다시 실행하면 새 토큰이 생기고, 기존 토큰은 즉시 무효가 됩니다.
 */
function generateToken() {
  var chars = 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz23456789';
  var bytes = Utilities.getUuid().replace(/-/g, '') + Utilities.getUuid().replace(/-/g, '');
  var t = '';
  for (var i = 0; i < 40; i++) {
    t += chars.charAt(parseInt(bytes.charAt(i), 16) * 3 + (i % 3));
  }
  PropertiesService.getScriptProperties().setProperty(TOKEN_PROP, t);
  showTokenDialog_(t, '새 접근 토큰이 생성되었습니다.\n앱의 "접근 토큰" 칸에 붙여넣으세요.\n\n기존 토큰은 더 이상 동작하지 않습니다.');
  return t;
}

/** 현재 토큰 확인 */
function showToken() {
  var t = PropertiesService.getScriptProperties().getProperty(TOKEN_PROP);
  if (!t) { SpreadsheetApp.getUi().alert('토큰이 없습니다. generateToken을 실행하세요.'); return; }
  showTokenDialog_(t, '현재 접근 토큰입니다.');
}

/** 토큰 검사를 끈다 (권장하지 않음) */
function disableToken() {
  PropertiesService.getScriptProperties().deleteProperty(TOKEN_PROP);
  SpreadsheetApp.getUi().alert('토큰 검사를 껐습니다. 주소를 아는 사람은 누구나 데이터를 읽을 수 있습니다.');
}

function showTokenDialog_(t, msg) {
  var html = '<div style="font-family:sans-serif;font-size:13px;line-height:1.6">'
    + msg.replace(/\n/g, '<br>')
    + '<div style="margin-top:12px"><input id="t" value="' + t + '" readonly '
    + 'style="width:100%;font-family:monospace;font-size:13px;padding:8px;'
    + 'border:1px solid #ccc;border-radius:4px" onclick="this.select()"></div>'
    + '<p style="color:#666">칸을 눌러 전체 선택한 뒤 복사하세요.</p></div>';
  SpreadsheetApp.getUi().showModalDialog(
    HtmlService.createHtmlOutput(html).setWidth(460).setHeight(210), '접근 토큰');
}

/** 상수 시간에 가깝게 비교 */
function tokenOk_(given) {
  var need = PropertiesService.getScriptProperties().getProperty(TOKEN_PROP);
  if (!need) return true;                 // 토큰 미설정 시 검사 생략
  given = String(given || '');
  if (given.length !== need.length) return false;
  var diff = 0;
  for (var i = 0; i < need.length; i++) diff |= given.charCodeAt(i) ^ need.charCodeAt(i);
  return diff === 0;
}

/** 관심목록 전체 갱신 (트리거용) */
function refreshWatchlist() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var list = watchList_(ss), out = [];
  for (var i = 0; i < list.length; i++) {
    try {
      var got = pullSeries_(ss, list[i].ticker || list[i].id, DEFAULT_YEARS);
      var n = mergeArchive_(ss, list[i].id, got.rows);
      watchUpsert_(ss, list[i].id, got.ticker, n);
      out.push(list[i].id + ': ' + n + '행');
    } catch (e) {
      out.push(list[i].id + ': 실패 — ' + e.message);
    }
  }
  Logger.log(out.join('\n'));
  ss.toast(out.join('\n') || '관심목록이 비어 있습니다', '갱신 결과', 20);
}

/* ═══════════════ 설정 동기화 ═══════════════ */

function configSheet_(ss) {
  var sh = ss.getSheetByName(CONFIG_SHEET);
  if (!sh) {
    sh = ss.insertSheet(CONFIG_SHEET);
    sh.getRange(1, 1, 1, 3).setValues([['갱신시각', '기기', '설정(JSON)']]).setFontWeight('bold');
    sh.setFrozenRows(1);
    sh.hideSheet();
  }
  return sh;
}
function configGet_(ss) {
  var sh = configSheet_(ss);
  if (sh.getLastRow() < 2) return null;
  return {
    at: sh.getRange(2, 1).getDisplayValue(),
    device: sh.getRange(2, 2).getValue(),
    data: sh.getRange(2, 3).getValue()
  };
}
function configSet_(ss, data, device) {
  var sh = configSheet_(ss);
  var now = Utilities.formatDate(new Date(), ss.getSpreadsheetTimeZone(), 'yyyy-MM-dd HH:mm:ss');
  sh.getRange(2, 1, 1, 3).setValues([[now, device || '', data]]);
  return now;
}

/* ═══════════════ 웹 앱 ═══════════════ */

/**
 * GET 파라미터
 *   mode=list                                  → 보관 중인 종목 목록
 *   mode=get   & ids=SOXL,QQQ                  → 아카이브를 압축 형식으로 반환
 *   mode=fetch & id=SOXL & ticker=NYSEARCA:SOXL & years=10
 *                                              → 새로 수집 후 병합·반환
 *   callback=fn                                → JSONP
 */
function doGet(e) {
  var p = (e && e.parameter) || {};
  var res = { ok: true, at: new Date().toISOString() };

  if (!tokenOk_(p.token)) {
    return reply_({ ok: false, error: '접근 토큰이 올바르지 않습니다' }, p.callback);
  }

  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var mode = p.mode || 'get';

    if (mode === 'list') {
      res.list = watchList_(ss).map(function (w) {
        var sh = ss.getSheetByName(ARCH_PREFIX + w.id);
        return { id: w.id, ticker: w.ticker, rows: sh ? Math.max(0, sh.getLastRow() - 1) : 0 };
      });

    } else if (mode === 'cfgget') {
      res.cfg = configGet_(ss);

    } else if (mode === 'cfgset') {
      if (!p.data) throw new Error('설정 데이터가 없습니다');
      res.at = configSet_(ss, p.data, p.device);

    } else if (mode === 'fetch') {
      var id = String(p.id || '').trim().toUpperCase();
      if (!id) throw new Error('id가 필요합니다');
      var got = pullSeries_(ss, p.ticker || id, parseFloat(p.years) || DEFAULT_YEARS);
      var n = mergeArchive_(ss, id, got.rows);
      watchUpsert_(ss, id, got.ticker, n);
      res.series = {};
      res.series[id] = encodeArchive_(ss, id);
      res.resolved = got.ticker;

    } else {
      var ids = String(p.ids || p.series || '').split(',')
        .map(function (x) { return x.trim().toUpperCase(); }).filter(String);
      if (!ids.length) ids = watchList_(ss).map(function (w) { return w.id; });
      res.series = {};
      ids.forEach(function (id) {
        try { res.series[id] = encodeArchive_(ss, id); }
        catch (err) { res.series[id] = { error: err.message }; }
      });
    }
  } catch (err) {
    res = { ok: false, error: err.message };
  }

  return reply_(res, p.callback);
}

function reply_(obj, callback) {
  var body = JSON.stringify(obj);
  if (callback) {
    return ContentService.createTextOutput(callback + '(' + body + ');')
      .setMimeType(ContentService.MimeType.JAVASCRIPT);
  }
  return ContentService.createTextOutput(body).setMimeType(ContentService.MimeType.JSON);
}

/** 아카이브 → {s:시작일, d:'간격,간격,...', c:'종가,종가,...'} */
function encodeArchive_(ss, id) {
  var sh = ss.getSheetByName(ARCH_PREFIX + id);
  if (!sh || sh.getLastRow() < 2) throw new Error('보관된 데이터가 없습니다');

  var vals = sh.getRange(2, 1, sh.getLastRow() - 1, 2).getValues();
  var tz = ss.getSpreadsheetTimeZone(), pairs = [];
  for (var i = 0; i < vals.length; i++) {
    var d = vals[i][0], v = parseFloat(vals[i][1]);
    if (d instanceof Date) d = Utilities.formatDate(d, tz, 'yyyy-MM-dd');
    d = String(d).slice(0, 10);
    if (/^\d{4}-\d{2}-\d{2}$/.test(d) && isFinite(v) && v > 0) pairs.push([d, v]);
  }
  pairs.sort(function (a, b) { return a[0] < b[0] ? -1 : a[0] > b[0] ? 1 : 0; });
  if (!pairs.length) throw new Error('유효한 행이 없습니다');

  var DAY = 86400000, deltas = [], closes = [];
  var prev = Date.parse(pairs[0][0] + 'T00:00:00Z');
  closes.push(String(Math.round(pairs[0][1] * 1e6) / 1e6));
  for (var k = 1; k < pairs.length; k++) {
    var t = Date.parse(pairs[k][0] + 'T00:00:00Z');
    deltas.push(Math.round((t - prev) / DAY));
    closes.push(String(Math.round(pairs[k][1] * 1e6) / 1e6));
    prev = t;
  }
  return { s: pairs[0][0], d: deltas.join(','), c: closes.join(',') };
}

/* ═══════════════ 시트 메뉴 ═══════════════ */

function onOpen() {
  SpreadsheetApp.getUi().createMenu('백테스터 데이터')
    .addItem('관심목록 전체 갱신', 'refreshWatchlist')
    .addItem('상태 보기', 'showStatus')
    .addSeparator()
    .addItem('접근 토큰 보기', 'showToken')
    .addItem('접근 토큰 재발급', 'generateToken')
    .addSeparator()
    .addItem('저장된 설정 지우기', 'clearConfig')
    .addToUi();
}
function clearConfig() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sh = ss.getSheetByName(CONFIG_SHEET);
  if (sh && sh.getLastRow() > 1) sh.deleteRow(2);
  SpreadsheetApp.getUi().alert('저장된 설정을 지웠습니다.');
}

function showStatus() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var list = watchList_(ss);
  if (!list.length) { SpreadsheetApp.getUi().alert('관심목록이 비어 있습니다.'); return; }
  var msg = list.map(function (w) {
    var sh = ss.getSheetByName(ARCH_PREFIX + w.id);
    if (!sh || sh.getLastRow() < 2) return w.id + ': 비어 있음';
    return w.id + ' (' + w.ticker + '): ' + (sh.getLastRow() - 1) + '행, 최종 '
      + sh.getRange(sh.getLastRow(), 1).getDisplayValue();
  });
  SpreadsheetApp.getUi().alert(msg.join('\n'));
}
