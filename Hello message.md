Hello message  
  
  
function main(workbook: ExcelScript.Workbook, csvText: string): string {  
  const rows: string[][] = [];  
  let row: string[] = [], field = "", inQuotes = false;  
  for (let i = 0; i < csvText.length; i++) {  
    const c = csvText[i];  
    if (inQuotes) {  
      if (c === '"' && csvText[i + 1] === '"') { field += '"'; i++; }  
      else if (c === '"') { inQuotes = false; }  
      else { field += c; }  
    } else {  
      if (c === '"') { inQuotes = true; }  
      else if (c === ",") { row.push(field); field = ""; }  
      else if (c === "\n" || c === "\r") {  
        if (c === "\r" && csvText[i + 1] === "\n") i++;  
        row.push(field); field = "";  
        if (row.length > 1 || row[0] !== "") rows.push(row);  
        row = [];  
      } else { field += c; }  
    }  
  }  
  if (field !== "" || row.length > 0) { row.push(field); rows.push(row); }  
  
  const headers = rows[0].map(h => h.trim());  
  const out: { [key: string]: string }[] = [];  
  for (let i = 1; i < rows.length; i++) {  
    const obj: { [key: string]: string } = {};  
    for (let j = 0; j < headers.length; j++) obj[headers[j]] = rows[i][j] ?? "";  
    out.push(obj);  
  }  
  return JSON.stringify(out);  
}  
