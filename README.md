<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1.0"/>
<title>UberEats 業務追蹤系統</title>

<!-- PWA 手機 App 屬性設定 -->
<meta name="theme-color" content="#06C167"/>
<meta name="apple-mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent"/>
<meta name="apple-mobile-web-app-title" content="UE追蹤"/>
<link rel="manifest" href='data:application/json,{"name":"UberEats 業務追蹤系統","short_name":"UE追蹤","start_url":".","display":"standalone","background_color":"#f5f5f5","theme_color":"#06C167","icons":[{"src":"https://cdn-icons-png.flaticon.com/512/857/857681.png","sizes":"512x512","type":"image/png"}]}'/>

<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:'PingFang TC','Microsoft JhengHei','Segoe UI',sans-serif;background:#f5f5f5;color:#1a1a1a;font-size:13px;padding:12px}
::-webkit-scrollbar{width:6px;height:6px}
::-webkit-scrollbar-track{background:#f1f1f1}
::-webkit-scrollbar-thumb{background:#ccc;border-radius:3px}

/* 手機版排版優化 */
.container{max-width:1400px;margin:0 auto;background:#fff;padding:15px;border-radius:8px;box-shadow:0 2px 5px rgba(0,0,0,0.05)}
.title-bar{color:#06C167;margin-bottom:15px;display:flex;align-items:center;gap:6px;font-size:18px;font-weight:bold}
.table-wrapper{overflow-x:auto;border:1px solid #ddd;border-radius:6px}
table{width:100%;border-collapse:collapse;text-align:left}
th,td{padding:10px 12px;border-bottom:1px solid #eeeeee;white-space:nowrap}
th{background:#f7f7f7;font-weight:bold;position:sticky;top:0;z-index:10}
tr:hover{background:#fafafa}
</style>
</head>
<body>
<div id="root"></div>

<!-- 離線 Service Worker 腳本，App 必備 -->
<script>
if ('serviceWorker' in navigator) {
  const swCode = `self.addEventListener("fetch",e=>{e.respondWith(fetch(e.request).catch(()=>caches.match(e.request)));});`;
  const blob = new Blob([swCode], { type: "application/javascript" });
  navigator.serviceWorker.register(URL.createObjectURL(blob));
}
</script>

<script type="text/babel">
const {useState,useEffect,useCallback,useRef} = React;

// ── 工具 ──────────────────────────────────────────────────────────────────────
const today = new Date(); today.setHours(0,0,0,0);
const C = {
  black:"#000",green:"#06C167",greenDark:"#05a557",greenLight:"#e6f9f0",
  warn:"#ff6b00",warnLight:"#fff3e8",danger:"#e53935",dangerLight:"#fdecea",
  info:"#1976d2",infoLight:"#e3f0fb",
  g100:"#f7f7f7",g200:"#eeeeee",g300:"#dddddd",g500:"#999",g700:"#555",
};
function parseD(s){if(!s)return null;const d=new Date(s);d.setHours(0,0,0,0);return isNaN(d)?null:d;}
function toISO(d){return d?d.toISOString().split("T")[0]:null;}
function fmtD(s){if(!s)return"—";const d=parseD(s);if(!d)return s;return`${d.getMonth()+1}/${d.getDate()}`;}
function addWD(d,n){let dt=new Date(d);let a=0;while(a<n){dt.setDate(dt.getDate()+1);const w=dt.getDay();if(w!==0&&w!==6)a++;}return dt;}

// ── 流程 ──────────────────────────────────────────────────────────────────────
function getSteps(s){
  const cs=parseD(s.contractSent),csig=parseD(s.contractSigned);
  const menu=parseD(s.menuDone),photo=parseD(s.photoDone);
  const amlU=parseD(s.amlUpload),amlD=parseD(s.amlDone);
  const tab=parseD(s.tabletSent),live=parseD(s.liveDate);
  const csDue=cs?new Date(cs.getTime()+7*864e5):null;
  const menuDue=csig?addWD(csig,3):null,photoDue=csig?addWD(csig,3):null;
  const amlDue=amlU?addWD(amlU,3):null;
  const pre=[amlD,menu,photo].filter(Boolean);
  const lastPre=pre.length?new Date(Math.max(...pre.map(d=>d.getTime()))):null;
  const tabDue=lastPre?addWD(lastPre,4):null;
  const st=(done,due,active)=>{if(done)return"done";if(!active)return"pending";if(due&&today>due)return"overdue";return"active";};
  return[
    {key:"contract_sent",icon:"📄",label:"合約寄出",doneDate:cs,dueDate:null,status:cs?"done":"pending"},
    {key:"contract_signed",icon:"✍️",label:"已回簽",doneDate:csig,dueDate:csDue,status:st(csig,csDue,cs)},
    {key:"menu",icon:"🍽",label:"菜單完成",doneDate:menu,dueDate:menuDue,status:st(menu,menuDue,csig)},
    {key:"photo",icon:"📸",label:"拍攝完成",doneDate:photo,dueDate:photoDue,status:st(photo,photoDue,csig)},
    {key:"aml",icon:"🔐",label:"AML通過",doneDate:amlD,dueDate:amlDue,status:st(amlD,amlDue,amlU)},
    {key:"tablet",icon:"📦",label:"平板寄出",doneDate:tab,dueDate:tabDue,status:st(tab,tabDue,lastPre)},
    {key:"live",icon:"🟢",label:"已開通",doneDate:live,dueDate:null,status:live?"done":tab?"active":"pending"},
  ];
}
function isOverdue(s){return getSteps(s).some(x=>x.status==="overdue");}
function contractDays(s){
  if(!s.contractSent||s.contractSigned)return null;
  return Math.floor((today-parseD(s.contractSent))/864e5);
}

// ── 範例資料 ──────────────────────────────────────────────────────────────────
const SAMPLE=[
  {name:"澄味堂豆乳雞",tag:"6月ft",contractSent:"2026-04-21",contractSigned:"2026-04-28",menuDone:"2026-05-03",photoDone:"",amlUpload:"2026-05-04",amlStatus:"manual_approval",amlDone:"",missing:"缺照片",cw:"eng-SF51905995",tabletSent:"",tabletReceived:"",liveDate:"",eta:"缺照片",notes:"",opp:"",photoDate:""},
  {name:"棠以商行-日式鹽滷豆花",tag:"6月ft",contractSent:"2026-04-18",contractSigned:"2026-04-25",menuDone:"2026-05-01",photoDone:"2026-05-15",amlUpload:"2026-05-11",amlStatus:"manual_approval",amlDone:"2026-05-18",missing:"",cw:"eng-SF51911810",tabletSent:"2026-05-22",tabletReceived:"2026-05-25",liveDate:"2026-05-28",eta:"已通",notes:"",opp:"",photoDate:""},
  {name:"里好餐桌",tag:"6月ft",contractSent:"2026-05-19",contractSigned:"2026-05-26",menuDone:"",photoDone:"",amlUpload:"2026-05-22",amlStatus:"manual_approval",amlDone:"",missing:"傳的銀行帳戶名稱",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"月中",notes:"",opp:"",photoDate:""},
  {name:"小豆堡",tag:"6月ft",contractSent:"2026-05-17",contractSigned:"2026-05-24",menuDone:"2026-05-29",photoDone:"",amlUpload:"2026-05-22",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"隨時",notes:"",opp:"",photoDate:""},
  {name:"咖啡施",tag:"6月ft",contractSent:"2026-05-18",contractSigned:"",menuDone:"",photoDone:"",amlUpload:"",amlStatus:"",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"泰泰喜歡",tag:"6月ft",contractSent:"2026-05-19",contractSigned:"2026-05-26",menuDone:"",photoDone:"",amlUpload:"2026-05-22",amlStatus:"identity_not_verified",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"忠孝夜市|烤霸傳統生烤玉米",tag:"6月ft",contractSent:"2026-05-26",contractSigned:"2026-06-02",menuDone:"",photoDone:"",amlUpload:"2026-05-26",amlStatus:"identity_not_verified",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"6/15",notes:"",opp:"",photoDate:""},
  {name:"九丹煮奶-台南國華店",tag:"6月ft",contractSent:"2026-06-01",contractSigned:"2026-06-08",menuDone:"",photoDone:"",amlUpload:"2026-06-05",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"NOVO-UETW201",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"鶴立雞群 HO-LI 炸雞",tag:"6月ft",contractSent:"2026-06-03",contractSigned:"2026-06-10",menuDone:"",photoDone:"2026-06-04",amlUpload:"2026-06-04",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"NOVO-UETW205",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"YaLo鴨嚼",tag:"6月ft",contractSent:"2026-06-04",contractSigned:"2026-06-11",menuDone:"",photoDone:"2026-06-11",amlUpload:"2026-06-13",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"shunfeng-SF519",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"洋行炸雞 國華店",tag:"6月ft sunmi",contractSent:"2026-06-12",contractSigned:"",menuDone:"",photoDone:"",amlUpload:"2026-06-13",amlStatus:"identity_not_verified",amlDone:"",missing:"",cw:"SF51957552555",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"洋行炸雞 榮譽店",tag:"6月ft sunmi(推薦)",contractSent:"2026-06-12",contractSigned:"",menuDone:"",photoDone:"",amlUpload:"2026-06-13",amlStatus:"identity_not_verified",amlDone:"",missing:"",cw:"SF51958640065",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"洋行炸雞 安平店",tag:"6月ft(推薦)",contractSent:"2026-06-12",contractSigned:"",menuDone:"",photoDone:"",amlUpload:"2026-06-13",amlStatus:"identity_not_verified",amlDone:"",missing:"",cw:"SF022-",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"林香檸手打檸檬茶-一中店",tag:"6月ft",contractSent:"2026-06-01",contractSigned:"2026-06-08",menuDone:"",photoDone:"2026-06-05",amlUpload:"2026-06-04",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"shunfeng-SF513-",tabletSent:"",tabletReceived:"",liveDate:"",eta:"月中",notes:"",opp:"",photoDate:""},
  {name:"念水町|串燒|關東煮|炸物",tag:"6月ft",contractSent:"2026-06-06",contractSigned:"2026-06-13",menuDone:"",photoDone:"2026-06-08",amlUpload:"2026-06-13",amlStatus:"manual_approval",amlDone:"",missing:"",cw:"shunfeng-SF519-",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"石原省太郎",tag:"違規",contractSent:"2026-06-02",contractSigned:"2026-06-09",menuDone:"",photoDone:"",amlUpload:"",amlStatus:"",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
  {name:"Q脆|香港雞蛋仔|中西區必吃美食",tag:"違規",contractSent:"2026-06-04",contractSigned:"",menuDone:"",photoDone:"",amlUpload:"",amlStatus:"",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:"",opp:"",photoDate:""},
];

const EMPTY={name:"",tag:"",opp:"",contractSent:"",contractSigned:"",menuDone:"",photoDate:"",photoDone:"",amlUpload:"",amlStatus:"",amlDone:"",missing:"",cw:"",tabletSent:"",tabletReceived:"",liveDate:"",eta:"",notes:""};

// ── 小元件 ────────────────────────────────────────────────────────────────────
function Badge({type="gray",children}){
  const map={green:[C.greenLight,"#059a52"],warn:[C.warnLight,C.warn],danger:[C.dangerLight,C.danger],info:[C.infoLight,C.info],gray:[C.g200,C.g700]};
  const[bg,color]=map[type]||map.gray;
  return <span style={{display:"inline-block",padding:"2px 8px",borderRadius:20,fontSize:10,fontWeight:700,background:bg,color,whiteSpace:"nowrap"}}>{children}</span>;
}

function Pipeline({steps}){
  const dotColors={done:[C.green,"#000"],active:[C.warn,"#fff"],overdue:[C.danger,"#fff"],pending:[C.g200,C.g500]};
  const lblColors={done:C.green,active:C.warn,overdue:C.danger,pending:C.g500};
  return(
    <div style={{display:"flex",alignItems:"flex-start",gap:2,minWidth:440}}>
      {steps.map((st,i)=>{
        const[bg,color]=dotColors[st.status]||dotColors.pending;
        return (
          <React.Fragment key={st.key}>
            {i>0&&<div style={{height:2,flex:1,background:steps[i-1].status==="done"?C.green:C.g300,marginTop:12,minWidth:4}}/>}
            <div style={{display:"flex",flexDirection:"column",alignItems:"center",gap:1}}>
              <div title={`${st.label}${st.dueDate?" 期限:"+fmtD(toISO(st.dueDate)):""}${st.doneDate?" 完成:"+fmtD(toISO(st.doneDate)):""}`}
                style={{width:24,height:24,borderRadius:"50%",background:bg,color,display:"flex",alignItems:"center",justifyContent:"center",fontSize:12,boxShadow:"0 1px 3px rgba(0,0,0,0.1)"}}>
                {st.icon}
              </div>
              <span style={{fontSize:10,color:lblColors[st.status],whiteSpace:"nowrap",fontWeight:st.status==="active"||st.status==="overdue"?"bold":"normal"}}>
                {st.label}
              </span>
              {st.doneDate && <span style={{fontSize:9,color:C.g500}}>{fmtD(toISO(st.doneDate))}</span>}
            </div>
          </React.Fragment>
        );
      })}
    </div>
  );
}

// ── 主應用主畫面元件 ──────────────────────────────────────────────────────────
function App() {
  const [data] = useState(SAMPLE);

  return (
    <div className="container">
      <div className="title-bar">
        <span>🟢</span> UberEats 業務追蹤系統
      </div>
      <div className="table-wrapper">
        <table>
          <thead>
            <tr>
              <th>店家名稱</th>
              <th>分類標籤</th>
              <th>追蹤進度流程線</th>
              <th>缺件/備註說明</th>
              <th>預計開通 (ETA)</th>
              <th>CW 案號</th>
            </tr>
          </thead>
          <tbody>
            {data.map((item, idx) => {
              const steps = getSteps(item);
              const overdue = isOverdue(item);
              return (
                <tr key={idx} style={{background: overdue ? "#fff9f9" : "inherit"}}>
                  <td style={{fontWeight:"bold",color:"#222"}}>{item.name}</td>
                  <td>
                    <Badge type={item.tag === "違規" ? "danger" : "info"}>
                      {item.tag}
                    </Badge>
                  </td>
                  <td><Pipeline steps={steps} /></td>
                  <td style={{color: C.danger,fontWeight:"bold"}}>{item.missing || "—"}</td>
                  <td><Badge type={item.eta && item.eta !== "已通" ? "warn" : "green"}>{item.eta || "—"}</Badge></td>
                  <td style={{fontFamily:"monospace",color:"#666",fontSize:12}}>{item.cw || "—"}</td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
}

// ── 渲染至實體 DOM ────────────────────────────────────────────────────────────
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
</script>
</body>
</html>
