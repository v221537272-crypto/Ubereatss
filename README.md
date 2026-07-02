<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1.0"/>
<title>UberEats 業務追蹤系統</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.2/babel.min.js"></script>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:'PingFang TC','Microsoft JhengHei','Segoe UI',sans-serif;background:#f5f5f5;color:#1a1a1a;font-size:13px}
::-webkit-scrollbar{width:6px;height:6px}
::-webkit-scrollbar-track{background:#f1f1f1}
::-webkit-scrollbar-thumb{background:#ccc;border-radius:3px}
</style>
</head>
<body>
<div id="root"></div>
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
        return(
          <React.Fragment key={st.key}>
            {i>0&&<div style={{height:2,flex:1,background:steps[i-1].status==="done"?C.green:C.g300,marginTop:12,minWidth:4}}/>}
            <div style={{display:"flex",flexDirection:"column",alignItems:"center",gap:1}}>
              <div title={`${st.label}${st.dueDate?" 期限:"+fmtD(toISO(st.dueDate)):""}${st.doneDate?" 完成:"+fmtD(toISO(st.doneDate)):""}`}
                style={{width:24,height:24,borderRadius:"50%",background:bg,color,display:"flex",alignItems:"center",justifyContent:"center",fontSize:10,fontWeight:800,border:`2px solid ${bg}`,boxShadow:st.status==="overdue"?"0 0 0 3px rgba(229,57,53,.25)":st.status==="active"?"0 0 0 3px rgba(255,107,0,.2)":"none"}}>
                {st.status==="done"?"✓":i+1}
              </div>
              <div style={{fontSize:8.5,color:lblColors[st.status]||C.g500,textAlign:"center",whiteSpace:"nowrap",fontWeight:st.status!=="pending"?700:400,lineHeight:1.2}}>{st.label}</div>
              <div style={{fontSize:7.5,color:C.g500,textAlign:"center",lineHeight:1.2}}>{st.doneDate?fmtD(toISO(st.doneDate)):st.dueDate&&st.status!=="done"?"期:"+fmtD(toISO(st.dueDate)):""}</div>
            </div>
          </React.Fragment>
        );
      })}
    </div>
  );
}

const inp={border:`1.5px solid ${C.g300}`,borderRadius:7,padding:"7px 10px",fontSize:12,fontFamily:"inherit",background:C.g100,width:"100%",outline:"none"};
const SecTitle=({t})=><div style={{gridColumn:"1/-1",borderTop:`1.5px solid ${C.g200}`,paddingTop:12,fontSize:11,fontWeight:800,color:C.g500,letterSpacing:".05em",textTransform:"uppercase",marginTop:4}}>{t}</div>;
const FF=({label,children})=><div style={{display:"flex",flexDirection:"column",gap:4}}><label style={{fontSize:11,fontWeight:700,color:C.g700}}>{label}</label>{children}</div>;

// ── 主程式 ────────────────────────────────────────────────────────────────────
function App(){
  const[stores,setStores]=useState([]);
  const[loading,setLoading]=useState(true);
  const[saving,setSaving]=useState(false);
  const[view,setView]=useState("list");
  const[search,setSearch]=useState("");
  const[filterStep,setFilterStep]=useState("");
  const[filterWarn,setFilterWarn]=useState("");
  const[modal,setModal]=useState(null);
  const[calYM,setCalYM]=useState([today.getFullYear(),today.getMonth()]);
  const[toast,setToast]=useState(null);

  const KEY="ubereats_stores_v3";

  // 資料存在 localStorage（多人可各自匯入同份資料）
  useEffect(()=>{
    try{
      const raw=localStorage.getItem(KEY);
      setStores(raw?JSON.parse(raw):SAMPLE);
    }catch{setStores(SAMPLE);}
    setLoading(false);
  },[]);

  const saveStores=async(next)=>{
    setStores(next);setSaving(true);
    try{localStorage.setItem(KEY,JSON.stringify(next));}catch(e){console.error(e);}
    setSaving(false);
  };

  const showToast=(msg)=>{setToast(msg);setTimeout(()=>setToast(null),2500);};

  // ── 統計 ──────────────────────────────────────────────────────────────────
  const m=today.getMonth(),yr=today.getFullYear();
  const fw=new Date(yr,m,14);
  const stats={
    total:stores.length,
    live:stores.filter(s=>s.liveDate).length,
    contract:stores.filter(s=>s.contractSent&&!s.contractSigned).length,
    inprog:stores.filter(s=>s.contractSigned&&!s.liveDate).length,
    overdue:stores.filter(isOverdue).length,
    goalDone:stores.filter(s=>{const d=parseD(s.tabletReceived||s.liveDate);return d&&d.getMonth()===m&&d.getFullYear()===yr&&d<=fw;}).length,
  };

  // ── 篩選 ──────────────────────────────────────────────────────────────────
  const filtered=stores.map((s,i)=>({s,i})).filter(({s})=>{
    if(search&&!s.name.toLowerCase().includes(search.toLowerCase()))return false;
    if(filterStep){
      const steps=getSteps(s);
      const idx={contract_sent:0,contract_signed:1,menu:2,photo:3,aml:4,tablet:5,live:6}[filterStep];
      if(idx!==undefined){const st=steps[idx];if(filterStep==="live"&&st.status!=="done")return false;if(filterStep!=="live"&&st.status!=="active"&&st.status!=="overdue")return false;}
    }
    if(filterWarn==="overdue"&&!isOverdue(s))return false;
    if(filterWarn==="7d"){const d=contractDays(s);if(d===null||d>7)return false;}
    return true;
  });

  // ── Modal ──────────────────────────────────────────────────────────────────
  const openAdd=()=>setModal({mode:"add",data:{...EMPTY}});
  const openEdit=(i)=>setModal({mode:"edit",idx:i,data:{...stores[i]}});
  const closeModal=()=>setModal(null);
  const setF=(k,v)=>setModal(m=>({...m,data:{...m.data,[k]:v}}));
  const saveModal=async()=>{
    if(!modal.data.name.trim()){alert("請填寫店名！");return;}
    const next=[...stores];
    if(modal.mode==="add")next.unshift(modal.data);
    else next[modal.idx]=modal.data;
    await saveStores(next);closeModal();
    showToast(modal.mode==="add"?"✅ 店家已新增！":"✅ 資料已儲存！");
  };
  const delStore=async(i)=>{
    if(!confirm(`確定刪除「${stores[i].name}」？`))return;
    await saveStores(stores.filter((_,idx)=>idx!==i));showToast("🗑 已刪除");
  };

  // ── 匯出/匯入 ─────────────────────────────────────────────────────────────
  const exportCSV=()=>{
    const header="店名,合約寄出,回簽日期,菜單完成,拍攝完成,AML上傳,AML狀態,AML完成,缺件,CW編號,平板寄出,平板收到,開通日,預計上線,注意,備註";
    const rows=stores.map(s=>[s.name,s.contractSent,s.contractSigned,s.menuDone,s.photoDone,s.amlUpload,s.amlStatus,s.amlDone,s.missing,s.cw,s.tabletSent,s.tabletReceived,s.liveDate,s.eta,s.tag,s.notes].map(v=>`"${(v||"").replace(/"/g,'""')}"`).join(","));
    const blob=new Blob(["\uFEFF"+header+"\n"+rows.join("\n")],{type:"text/csv;charset=utf-8"});
    const a=document.createElement("a");a.href=URL.createObjectURL(blob);a.download="UberEats店家資料.csv";a.click();
  };

  const importCSV=(e)=>{
    const f=e.target.files[0];if(!f)return;
    const reader=new FileReader();
    reader.onload=ev=>{
      const lines=ev.target.result.replace(/\r/g,"").split("\n").filter(l=>l.trim());
      if(lines.length<2){alert("CSV格式有誤");return;}
      const newS=[];
      for(let i=1;i<lines.length;i++){
        const c=lines[i].split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/).map(v=>v.replace(/^"|"$/g,"").replace(/""/g,'"'));
        if(!c[0])continue;
        newS.push({name:c[0]||"",contractSent:c[1]||"",contractSigned:c[2]||"",menuDone:c[3]||"",photoDone:c[4]||"",amlUpload:c[5]||"",amlStatus:c[6]||"",amlDone:c[7]||"",missing:c[8]||"",cw:c[9]||"",tabletSent:c[10]||"",tabletReceived:c[11]||"",liveDate:c[12]||"",eta:c[13]||"",tag:c[14]||"",notes:c[15]||"",opp:"",photoDate:""});
      }
      if(!newS.length){alert("沒有讀取到資料");return;}
      if(confirm(`讀取到 ${newS.length} 筆，是否合併？（取消=覆蓋全部）`))saveStores([...stores,...newS]);
      else saveStores(newS);
      showToast(`✅ 匯入 ${newS.length} 筆！`);
    };
    reader.readAsText(f,"UTF-8");
    e.target.value="";
  };

  // ── 月曆 ──────────────────────────────────────────────────────────────────
  const[cY,cM]=calYM;
  const calData=()=>{
    const first=new Date(cY,cM,1).getDay(),days=new Date(cY,cM+1,0).getDate(),prev=new Date(cY,cM,0).getDate();
    const ev={};
    const add=(ds,e)=>{const d=parseD(ds);if(!d||d.getMonth()!==cM||d.getFullYear()!==cY)return;const k=d.getDate();if(!ev[k])ev[k]=[];ev[k].push(e);};
    stores.forEach(s=>{
      if(s.liveDate)add(s.liveDate,{label:"🟢 "+s.name,t:"green"});
      if(s.tabletSent)add(s.tabletSent,{label:"📦 "+s.name,t:"info"});
      if(s.photoDone)add(s.photoDone,{label:"📸 "+s.name,t:"warn"});
      if(s.contractSigned)add(s.contractSigned,{label:"✍️ "+s.name,t:"warn"});
    });
    return{first,days,prev,ev};
  };

  // ── GANTT ──────────────────────────────────────────────────────────────────
  const GanttView=()=>{
    const dim=new Date(yr,m+1,0).getDate(),COL=25,SW=148;
    const dayOf=s=>{const d=parseD(s);return d&&d.getMonth()===m&&d.getFullYear()===yr?d.getDate()-1:-1;};
    const BC={contract:"#06C167",menu:"#1976d2",photo:"#ff6b00",aml:"#7b1fa2",tablet:"#e53935"};
    const Bar=({start,end,color,label,top})=>{
      if(start<0&&end<0)return null;
      const s2=Math.max(0,start<0?end:start),e2=Math.min(dim-1,end<0?start:end);
      if(s2>dim-1||e2<0)return null;
      return<div style={{position:"absolute",left:SW+s2*COL+1,width:(e2-s2+1)*COL-2,height:13,top,background:color,borderRadius:3,display:"flex",alignItems:"center",padding:"0 3px",fontSize:8,fontWeight:700,color:"#fff",whiteSpace:"nowrap",overflow:"hidden",zIndex:2}}>{label}</div>;
    };
    return(
      <div style={{background:"#fff",borderRadius:10,boxShadow:"0 2px 12px rgba(0,0,0,.08)",overflow:"hidden"}}>
        <div style={{padding:"10px 14px",borderBottom:`1px solid ${C.g200}`,display:"flex",gap:14,flexWrap:"wrap",fontSize:11}}>
          {Object.entries({"合約":BC.contract,"菜單":BC.menu,"拍攝":BC.photo,"AML":BC.aml,"平板":BC.tablet}).map(([k,c])=>(
            <span key={k} style={{display:"flex",alignItems:"center",gap:4}}><span style={{width:10,height:10,background:c,borderRadius:2,display:"inline-block"}}/>{k}</span>
          ))}
        </div>
        <div style={{overflowX:"auto"}}>
          <div style={{minWidth:SW+dim*COL}}>
            <div style={{display:"flex",background:C.black,color:"#fff",fontSize:10,fontWeight:700}}>
              <div style={{width:SW,flexShrink:0,padding:"8px 10px",borderRight:"1px solid rgba(255,255,255,.1)"}}>店名</div>
              {Array.from({length:dim},(_,d)=>{
                const dd=new Date(yr,m,d+1),isT=dd.getTime()===today.getTime(),isW=dd.getDay()===0||dd.getDay()===6;
                return<div key={d} style={{width:COL,flexShrink:0,textAlign:"center",padding:"8px 0",color:isT?C.green:isW?"rgba(255,255,255,.3)":"rgba(255,255,255,.7)"}}>{d+1}</div>;
              })}
            </div>
            {stores.length===0&&<div style={{textAlign:"center",padding:40,color:C.g500}}>📊 尚無資料</div>}
            {stores.map((s,si)=>{
              const cs=dayOf(s.contractSent),csig=dayOf(s.contractSigned),menu=dayOf(s.menuDone),photo=dayOf(s.photoDone),amlD=dayOf(s.amlDone),tab=dayOf(s.tabletSent);
              return(
                <div key={si} style={{position:"relative",height:48,background:si%2?C.g100:"#fff",borderBottom:`1px solid ${C.g200}`,display:"flex",alignItems:"center"}}>
                  <div style={{width:SW,flexShrink:0,padding:"0 8px",fontSize:11,fontWeight:700,borderRight:`1px solid ${C.g200}`,height:"100%",display:"flex",alignItems:"center",overflow:"hidden",whiteSpace:"nowrap",textOverflow:"ellipsis"}}>{s.name}</div>
                  <div style={{flex:1,position:"relative",height:"100%"}}>
                    {Array.from({length:dim},(_,d)=>{
                      const dd=new Date(yr,m,d+1),isT=dd.getTime()===today.getTime(),isW=dd.getDay()===0||dd.getDay()===6;
                      return<div key={d} style={{position:"absolute",left:d*COL,width:COL,height:"100%",background:isT?"rgba(6,193,103,.07)":isW?"rgba(0,0,0,.025)":"transparent",borderRight:`1px solid ${C.g200}`}}/>;
                    })}
                    <Bar start={cs} end={csig} color={BC.contract} label="合約" top={7}/>
                    {menu>=0&&<Bar start={menu} end={menu} color={BC.menu} label="菜" top={7}/>}
                    {photo>=0&&<Bar start={photo} end={photo} color={BC.photo} label="拍" top={28}/>}
                    {amlD>=0&&<Bar start={amlD} end={amlD} color={BC.aml} label="AML" top={28}/>}
                    {tab>=0&&<Bar start={tab} end={tab} color={BC.tablet} label="平板" top={7}/>}
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      </div>
    );
  };

  if(loading)return<div style={{display:"flex",alignItems:"center",justifyContent:"center",height:"100vh",flexDirection:"column",gap:12}}><div style={{fontSize:40}}>🍔</div><div style={{fontWeight:700,color:C.g700}}>載入中…</div></div>;

  const{first,days,prev,ev}=calData();
  const evC={green:C.green,warn:C.warn,info:C.info};

  return(
    <div style={{fontFamily:"'PingFang TC','Microsoft JhengHei','Segoe UI',sans-serif",fontSize:13,color:"#1a1a1a",background:"#f5f5f5",minHeight:"100vh"}}>

      {/* Toast */}
      {toast&&<div style={{position:"fixed",top:14,right:14,background:C.black,color:"#fff",padding:"10px 18px",borderRadius:8,fontWeight:700,fontSize:13,zIndex:9999,boxShadow:"0 4px 20px rgba(0,0,0,.3)"}}>{toast}</div>}

      {/* Header */}
      <div style={{background:C.black,color:"#fff",padding:"0 22px",height:50,display:"flex",alignItems:"center",justifyContent:"space-between",position:"sticky",top:0,zIndex:200,boxShadow:"0 2px 8px rgba(0,0,0,.3)"}}>
        <div style={{display:"flex",alignItems:"center",gap:10,fontWeight:800,fontSize:14}}>
          <div style={{width:26,height:26,background:C.green,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",fontSize:10,fontWeight:900,color:"#000"}}>UE</div>
          UberEats 業務追蹤
          {saving&&<span style={{fontSize:11,opacity:.5,fontWeight:400}}>儲存中…</span>}
        </div>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <span style={{background:C.green,color:"#000",fontWeight:800,fontSize:11,padding:"3px 10px",borderRadius:20}}>{yr}年{m+1}月</span>
          <span style={{fontSize:11,opacity:.6}}>{today.toLocaleDateString("zh-TW",{month:"long",day:"numeric",weekday:"short"})}</span>
        </div>
      </div>

      {/* Nav */}
      <div style={{background:"#fff",borderBottom:`2px solid ${C.g200}`,display:"flex",padding:"0 22px",position:"sticky",top:50,zIndex:100}}>
        {[["list","📋 店家列表"],["gantt","📊 甘特圖"],["calendar","📅 月行事曆"]].map(([v,label])=>(
          <div key={v} onClick={()=>setView(v)} style={{padding:"12px 16px",fontSize:13,fontWeight:600,cursor:"pointer",borderBottom:`3px solid ${view===v?C.green:"transparent"}`,color:view===v?C.black:C.g500,marginBottom:-2,whiteSpace:"nowrap",userSelect:"none"}}>
            {label}
          </div>
        ))}
      </div>

      <div style={{maxWidth:1600,margin:"0 auto",padding:"16px 18px"}}>

        {/* Stats */}
        <div style={{display:"grid",gridTemplateColumns:"repeat(6,1fr)",gap:10,marginBottom:14}}>
          {[{n:stats.total,l:"本月總店家",b:C.black},{n:stats.live,l:"已開通",b:C.green},{n:stats.contract,l:"待回簽",b:C.warn},{n:stats.inprog,l:"進行中",b:C.info},{n:stats.overdue,l:"⚠️ 逾期",b:C.danger},{n:`${stats.goalDone}/16`,l:"月前半目標",b:C.green}].map((c,i)=>(
            <div key={i} style={{background:"#fff",borderRadius:10,padding:"12px 14px",boxShadow:"0 1px 6px rgba(0,0,0,.07)",borderTop:`3px solid ${c.b}`}}>
              <div style={{fontSize:24,fontWeight:800,lineHeight:1}}>{c.n}</div>
              <div style={{fontSize:10,color:C.g500,marginTop:3,fontWeight:600}}>{c.l}</div>
            </div>
          ))}
        </div>

        {/* Goal Banner */}
        <div style={{background:C.black,color:"#fff",borderRadius:10,padding:"12px 16px",marginBottom:14,display:"flex",alignItems:"center",gap:16,flexWrap:"wrap"}}>
          <div>
            <div style={{fontWeight:800}}>📦 本月前兩週目標</div>
            <div style={{fontSize:11,opacity:.7}}>截止日：{yr}/{m+1}/14｜需完成並收到平板 16 間</div>
          </div>
          <div style={{flex:1,background:"rgba(255,255,255,.15)",borderRadius:20,height:7,overflow:"hidden",minWidth:100,maxWidth:260}}>
            <div style={{width:`${Math.min(100,stats.goalDone/16*100)}%`,height:"100%",background:C.green,borderRadius:20}}/>
          </div>
          <div style={{fontSize:18,fontWeight:800,color:C.green,whiteSpace:"nowrap"}}>{stats.goalDone} / 16 間</div>
        </div>

        {/* ── LIST ── */}
        {view==="list"&&<>
          <div style={{display:"flex",gap:8,marginBottom:10,flexWrap:"wrap",alignItems:"center"}}>
            <input value={search} onChange={e=>setSearch(e.target.value)} placeholder="🔍 搜尋店名…" style={{...inp,width:190,margin:0}}/>
            <select value={filterStep} onChange={e=>setFilterStep(e.target.value)} style={{...inp,width:150,margin:0,cursor:"pointer"}}>
              <option value="">全部階段</option>
              <option value="contract_sent">合約已寄出</option>
              <option value="contract_signed">已回簽</option>
              <option value="menu">菜單製作中</option>
              <option value="photo">拍攝中</option>
              <option value="aml">AML審核中</option>
              <option value="tablet">平板寄出</option>
              <option value="live">已開通</option>
            </select>
            <select value={filterWarn} onChange={e=>setFilterWarn(e.target.value)} style={{...inp,width:130,margin:0,cursor:"pointer"}}>
              <option value="">全部狀態</option>
              <option value="overdue">⚠️ 逾期</option>
              <option value="7d">📌 合約7天內</option>
            </select>
            <div style={{marginLeft:"auto",display:"flex",gap:8,flexWrap:"wrap"}}>
              <label style={{padding:"7px 12px",borderRadius:7,border:`1.5px solid ${C.g300}`,background:"#fff",cursor:"pointer",fontSize:11,fontWeight:700}}>
                📥 匯入CSV<input type="file" accept=".csv" style={{display:"none"}} onChange={importCSV}/>
              </label>
              <button onClick={exportCSV} style={{padding:"7px 12px",borderRadius:7,border:`1.5px solid ${C.g300}`,background:"#fff",cursor:"pointer",fontSize:11,fontWeight:700}}>📤 匯出CSV</button>
              <button onClick={openAdd} style={{padding:"7px 14px",borderRadius:7,border:"none",background:C.green,color:"#000",cursor:"pointer",fontSize:13,fontWeight:800}}>＋ 新增店家</button>
            </div>
          </div>
          <div style={{background:"#fff",borderRadius:10,boxShadow:"0 1px 8px rgba(0,0,0,.08)",overflow:"hidden"}}>
            <div style={{overflowX:"auto"}}>
              <table style={{width:"100%",borderCollapse:"collapse",fontSize:11.5}}>
                <thead>
                  <tr>{["#","注意","店名","合約寄出","回簽日","流程進度（7步驟）","AML狀態","缺件","預計上線","操作"].map(h=><th key={h} style={{background:C.black,color:"#fff",padding:"9px 11px",textAlign:"left",fontWeight:700,whiteSpace:"nowrap"}}>{h}</th>)}</tr>
                </thead>
                <tbody>
                  {filtered.length===0&&<tr><td colSpan={10} style={{textAlign:"center",padding:40,color:C.g500}}>📭 沒有符合的店家</td></tr>}
                  {filtered.map(({s,i})=>{
                    const steps=getSteps(s),cd=contractDays(s),od=isOverdue(s);
                    const amlB=!s.amlStatus?<Badge type="gray">—</Badge>:s.amlStatus==="approved"?<Badge type="green">通過</Badge>:s.amlStatus==="rejected"?<Badge type="danger">拒絕</Badge>:s.amlStatus==="manual_approval"?<Badge type="warn">人工審</Badge>:<Badge type="danger">未驗證</Badge>;
                    return(
                      <tr key={i} style={{borderBottom:`1px solid ${C.g200}`,background:od?"#fff8f8":"#fff"}}>
                        <td style={{padding:"7px 11px",color:C.g500}}>{i+1}</td>
                        <td style={{padding:"7px 11px"}}>
                          {s.tag&&<div style={{fontSize:9,background:C.g200,color:C.g700,borderRadius:3,padding:"1px 4px",fontWeight:700,marginBottom:2,display:"inline-block"}}>{s.tag}</div>}
                          {cd!==null&&<div style={{fontSize:9,background:cd>7?C.dangerLight:C.warnLight,color:cd>7?C.danger:C.warn,borderRadius:3,padding:"1px 4px",fontWeight:700,display:"inline-block"}}>合約{cd}天{cd>7?"⚠️":""}</div>}
                          {od&&<div style={{fontSize:9,background:C.dangerLight,color:C.danger,borderRadius:3,padding:"1px 4px",fontWeight:700,marginTop:2,display:"inline-block"}}>逾期</div>}
                        </td>
                        <td style={{padding:"7px 11px"}}><div style={{fontWeight:700}}>{s.name}</div>{s.opp&&<a href={s.opp} target="_blank" rel="noreferrer" style={{fontSize:10,color:C.info}}>OPP↗</a>}</td>
                        <td style={{padding:"7px 11px",whiteSpace:"nowrap"}}>{fmtD(s.contractSent)}</td>
                        <td style={{padding:"7px 11px",whiteSpace:"nowrap"}}>{fmtD(s.contractSigned)}</td>
                        <td style={{padding:"7px 11px"}}><Pipeline steps={steps}/></td>
                        <td style={{padding:"7px 11px"}}>{amlB}</td>
                        <td style={{padding:"7px 11px",fontSize:11,color:s.missing?C.danger:C.g500}}>{s.missing||"—"}</td>
                        <td style={{padding:"7px 11px",fontSize:11}}>{s.eta||"—"}</td>
                        <td style={{padding:"7px 11px",whiteSpace:"nowrap"}}>
                          <button onClick={()=>openEdit(i)} style={{fontSize:11,padding:"3px 7px",borderRadius:5,border:`1px solid ${C.g300}`,background:"#fff",cursor:"pointer",marginRight:3,fontWeight:600}}>編輯</button>
                          <button onClick={()=>delStore(i)} style={{fontSize:11,padding:"3px 7px",borderRadius:5,border:"none",background:C.dangerLight,color:C.danger,cursor:"pointer",fontWeight:600}}>刪除</button>
                        </td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>
            </div>
          </div>
        </>}

        {/* ── GANTT ── */}
        {view==="gantt"&&<GanttView/>}

        {/* ── CALENDAR ── */}
        {view==="calendar"&&(
          <div style={{background:"#fff",borderRadius:10,boxShadow:"0 1px 8px rgba(0,0,0,.08)",padding:18}}>
            <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",marginBottom:14}}>
              <button onClick={()=>setCalYM([cM===0?cY-1:cY,cM===0?11:cM-1])} style={{padding:"6px 12px",borderRadius:7,border:`1.5px solid ${C.g300}`,background:"#fff",cursor:"pointer",fontWeight:700}}>◀ 上月</button>
              <div style={{fontWeight:800,fontSize:15}}>{cY}年{cM+1}月</div>
              <button onClick={()=>setCalYM([cM===11?cY+1:cY,cM===11?0:cM+1])} style={{padding:"6px 12px",borderRadius:7,border:`1.5px solid ${C.g300}`,background:"#fff",cursor:"pointer",fontWeight:700}}>下月 ▶</button>
            </div>
            <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",gap:4,marginBottom:4}}>
              {["日","一","二","三","四","五","六"].map(l=><div key={l} style={{textAlign:"center",fontSize:11,fontWeight:700,color:C.g500,padding:"3px 0"}}>{l}</div>)}
            </div>
            <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",gap:4}}>
              {Array.from({length:first},(_,i)=><div key={"p"+i} style={{background:C.g100,borderRadius:6,padding:5,minHeight:68,opacity:.4}}><div style={{fontWeight:700,fontSize:11}}>{prev-first+i+1}</div></div>)}
              {Array.from({length:days},(_,d)=>{
                const day=d+1,dd=new Date(cY,cM,day),isT=dd.getTime()===today.getTime(),isG=day<=14,evs=ev[day]||[];
                return(
                  <div key={day} style={{background:isG?"#e6f9f0":C.g100,borderRadius:6,padding:5,minHeight:68,border:isT?`2px solid ${C.green}`:"2px solid transparent"}}>
                    <div style={{fontWeight:700,fontSize:11,marginBottom:2,color:isT?C.green:"inherit"}}>{day}</div>
                    {evs.slice(0,3).map((e2,ei)=><div key={ei} style={{background:evC[e2.t]||C.green,color:e2.t==="green"?"#000":"#fff",borderRadius:3,padding:"1px 3px",fontSize:8.5,fontWeight:700,marginBottom:1,whiteSpace:"nowrap",overflow:"hidden",textOverflow:"ellipsis"}}>{e2.label}</div>)}
                    {evs.length>3&&<div style={{fontSize:8,color:C.g500}}>+{evs.length-3}筆</div>}
                  </div>
                );
              })}
            </div>
            <div style={{marginTop:10,padding:"8px 12px",background:C.greenLight,borderRadius:7,fontSize:11,color:"#059a52",fontWeight:700}}>📌 綠色底色 = 月份前兩週目標區間（需16間完成）</div>
          </div>
        )}
      </div>

      {/* ── MODAL ── */}
      {modal&&(
        <div onClick={e=>{if(e.target===e.currentTarget)closeModal();}} style={{position:"fixed",inset:0,background:"rgba(0,0,0,.5)",zIndex:500,display:"flex",alignItems:"center",justifyContent:"center",padding:14}}>
          <div style={{background:"#fff",borderRadius:14,width:"100%",maxWidth:600,maxHeight:"90vh",overflow:"hidden",display:"flex",flexDirection:"column",boxShadow:"0 8px 40px rgba(0,0,0,.25)"}}>
            <div style={{background:C.black,color:"#fff",padding:"14px 18px",display:"flex",alignItems:"center",justifyContent:"space-between",flexShrink:0}}>
              <div style={{fontWeight:800,fontSize:14}}>{modal.mode==="add"?"➕ 新增店家":"✏️ 編輯："+modal.data.name}</div>
              <button onClick={closeModal} style={{background:"none",border:"none",color:"#fff",fontSize:18,cursor:"pointer",opacity:.7}}>✕</button>
            </div>
            {modal.mode==="edit"&&(()=>{
              const steps=getSteps(modal.data);
              const sc={done:C.green,active:C.warn,overdue:C.danger,pending:C.g300};
              return(
                <div style={{padding:"12px 18px",borderBottom:`1px solid ${C.g200}`,background:C.g100,flexShrink:0}}>
                  <div style={{fontSize:10,fontWeight:800,color:C.g500,marginBottom:7,textTransform:"uppercase",letterSpacing:".05em"}}>流程進度</div>
                  <div style={{display:"flex",gap:5,flexWrap:"wrap"}}>
                    {steps.map(st=>(
                      <div key={st.key} style={{display:"flex",alignItems:"center",gap:4,padding:"4px 8px",borderRadius:5,background:"#fff",border:`1.5px solid ${sc[st.status]||C.g300}`,fontSize:10}}>
                        <span>{st.icon}</span>
                        <span style={{fontWeight:700,color:st.status==="done"?C.green:st.status==="overdue"?C.danger:st.status==="active"?C.warn:C.g500}}>{st.label}</span>
                        {st.doneDate&&<span style={{fontSize:9,color:C.g500}}>{fmtD(toISO(st.doneDate))}</span>}
                        {!st.doneDate&&st.dueDate&&<span style={{fontSize:9,color:C.g500}}>期:{fmtD(toISO(st.dueDate))}</span>}
                      </div>
                    ))}
                  </div>
                </div>
              );
            })()}
            <div style={{overflowY:"auto",padding:18,flex:1}}>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10}}>
                <div style={{gridColumn:"1/-1"}}><FF label="店名 *"><input style={inp} value={modal.data.name} onChange={e=>setF("name",e.target.value)} placeholder="店家名稱"/></FF></div>
                <FF label="OPP連結"><input style={inp} value={modal.data.opp||""} onChange={e=>setF("opp",e.target.value)} placeholder="https://…"/></FF>
                <FF label="注意事項"><select style={inp} value={modal.data.tag} onChange={e=>setF("tag",e.target.value)}><option value="">—</option>{["6月ft","6月ft sunmi","6月ft(推薦)","違規","其他"].map(t=><option key={t}>{t}</option>)}</select></FF>
                <SecTitle t="📅 合約流程"/>
                <FF label="合約寄出日期"><input type="date" style={inp} value={modal.data.contractSent} onChange={e=>setF("contractSent",e.target.value)}/></FF>
                <FF label="回簽日期"><input type="date" style={inp} value={modal.data.contractSigned} onChange={e=>setF("contractSigned",e.target.value)}/></FF>
                <SecTitle t="🍽 菜單 & 拍攝"/>
                <FF label="菜單完成日期"><input type="date" style={inp} value={modal.data.menuDone} onChange={e=>setF("menuDone",e.target.value)}/></FF>
                <FF label="預約拍攝時間"><input type="datetime-local" style={inp} value={modal.data.photoDate||""} onChange={e=>setF("photoDate",e.target.value)}/></FF>
                <FF label="拍攝完成日期"><input type="date" style={inp} value={modal.data.photoDone} onChange={e=>setF("photoDone",e.target.value)}/></FF>
                <SecTitle t="🔐 AML 審核"/>
                <FF label="資料上傳日期"><input type="date" style={inp} value={modal.data.amlUpload} onChange={e=>setF("amlUpload",e.target.value)}/></FF>
                <FF label="AML審核狀態"><select style={inp} value={modal.data.amlStatus} onChange={e=>setF("amlStatus",e.target.value)}><option value="">未上傳</option><option value="manual_approval">manual_approval</option><option value="identity_not_verified">identity_not_verified</option><option value="approved">approved（通過）</option><option value="rejected">rejected（拒絕）</option></select></FF>
                <FF label="AML完成日"><input type="date" style={inp} value={modal.data.amlDone} onChange={e=>setF("amlDone",e.target.value)}/></FF>
                <FF label="缺件說明"><input style={inp} value={modal.data.missing} onChange={e=>setF("missing",e.target.value)} placeholder="例：銀行帳戶封面"/></FF>
                <FF label="CW編號"><input style={inp} value={modal.data.cw} onChange={e=>setF("cw",e.target.value)} placeholder="shunfeng-SF519…"/></FF>
                <SecTitle t="📦 平板 & 開通"/>
                <FF label="平板寄出日期"><input type="date" style={inp} value={modal.data.tabletSent} onChange={e=>setF("tabletSent",e.target.value)}/></FF>
                <FF label="平板收到日期"><input type="date" style={inp} value={modal.data.tabletReceived} onChange={e=>setF("tabletReceived",e.target.value)}/></FF>
                <FF label="開通日期"><input type="date" style={inp} value={modal.data.liveDate} onChange={e=>setF("liveDate",e.target.value)}/></FF>
                <FF label="預計上線"><input style={inp} value={modal.data.eta} onChange={e=>setF("eta",e.target.value)} placeholder="月中/6/15/隨時"/></FF>
                <div style={{gridColumn:"1/-1"}}><FF label="備註"><textarea style={{...inp,minHeight:55,resize:"vertical"}} value={modal.data.notes} onChange={e=>setF("notes",e.target.value)} placeholder="其他說明…"/></FF></div>
              </div>
            </div>
            <div style={{padding:"12px 18px",borderTop:`1px solid ${C.g200}`,display:"flex",gap:8,justifyContent:"flex-end",flexShrink:0}}>
              <button onClick={closeModal} style={{padding:"8px 14px",borderRadius:7,border:`1.5px solid ${C.g300}`,background:"#fff",cursor:"pointer",fontWeight:700}}>取消</button>
              <button onClick={saveModal} style={{padding:"8px 18px",borderRadius:7,border:"none",background:C.green,color:"#000",cursor:"pointer",fontWeight:800,fontSize:13}}>💾 儲存</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<App/>);
</script>
</body>
</html>
