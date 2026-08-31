function main(workbook: ExcelScript.Workbook): string {  
  const sheet = workbook.getWorksheets()[0];  
  const range = sheet.getUsedRange();  
  const values = range.getValues();  
  const headers = values[0].map(h => String(h).trim());  
  const rows: { [key: string]: string }[] = [];  
  for (let i = 1; i < values.length; i++) {  
    const row: { [key: string]: string } = {};  
    for (let j = 0; j < headers.length; j++) {  
      row[headers[j]] = String(values[i][j] ?? "");  
    }  
    rows.push(row);  
  }  
  return JSON.stringify(rows);  
}  
  
  
[{"Job Code":"1234","Job Title":"Analyst","Purpose":"text","Education Requirements":"text","Duties and Responsibilities":"text"}]  
  
  
@equals(item()?['cr123_jobcode'], items('Apply_to_each_2')?['Job Code'])  
