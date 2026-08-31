# Candid   
  
function main(  
  workbook: ExcelScript.Workbook,  
  userInput: string,  
  itemsJson: string  
): string {  
  // itemsJson = JSON array from the flow: [{"id":"1","title":"BSA Monitr Anlyst","jobCode":"1234"}, ...]  
  const items: { id: string; title: string; jobCode: string }[] = JSON.parse(itemsJson);  
  
  // ---- Abbreviation/alias map: expand as you discover more ----  
  const aliases: { [key: string]: string } = {  
    "anlyst": "analyst", "anlst": "analyst",  
    "monitr": "monitoring", "mntr": "monitoring", "mon": "monitoring",  
    "mgr": "manager", "mgmt": "management",  
    "sr": "senior", "jr": "junior",  
    "assoc": "associate", "asst": "assistant",  
    "spec": "specialist", "splst": "specialist",  
    "coord": "coordinator", "coordntr": "coordinator",  
    "rep": "representative", "reprsntv": "representative",  
    "admin": "administrator", "admnstr": "administrator",  
    "eng": "engineer", "engr": "engineer",  
    "dev": "developer", "dvlpr": "developer",  
    "acct": "accounting", "acctnt": "accountant",  
    "hr": "human resources",  
    "vp": "vice president",  
    "dir": "director", "drctr": "director",  
    "ops": "operations", "oprtns": "operations",  
    "svc": "service", "svcs": "services",  
    "compl": "compliance", "cmplnc": "compliance",  
    "bus": "business", "biz": "business",  
    "dept": "department",  
    "tech": "technology", "technlgy": "technology",  
    "info": "information",  
    "sec": "security", "scrty": "security"  
  };  
  
  function normalize(text: string): string {  
    const words = text.toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter(w => w.length > 0);  
    return words.map(w => aliases[w] ?? w).join(" ");  
  }  
  
  function levenshtein(a: string, b: string): number {  
    const m = a.length, n = b.length;  
    if (m === 0) return n;  
    if (n === 0) return m;  
    let prev: number[] = [];  
    for (let j = 0; j <= n; j++) prev.push(j);  
    for (let i = 1; i <= m; i++) {  
      const curr: number[] = [i];  
      for (let j = 1; j <= n; j++) {  
        const cost = a[i - 1] === b[j - 1] ? 0 : 1;  
        curr.push(Math.min(curr[j - 1] + 1, prev[j] + 1, prev[j - 1] + cost));  
      }  
      prev = curr;  
    }  
    return prev[n];  
  }  
  
  function similarity(a: string, b: string): number {  
    const maxLen = Math.max(a.length, b.length);  
    if (maxLen === 0) return 100;  
    return Math.round((1 - levenshtein(a, b) / maxLen) * 100);  
  }  
  
  function tokenScore(inputNorm: string, titleNorm: string): number {  
    const inputTokens = inputNorm.split(" ");  
    const titleTokens = titleNorm.split(" ");  
    let matched = 0;  
    for (const it of inputTokens) {  
      let best = 0;  
      for (const tt of titleTokens) {  
        const s = similarity(it, tt);  
        if (s > best) best = s;  
      }  
      mat  
