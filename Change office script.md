# Change office script  
  
      for (let j = 0; j < headers.length; j++) obj[headers[j]] = rows[i][j] ?? "";  
  
  
      for (let j = 0; j < headers.length; j++) {  
        obj[headers[j]] = (rows[i][j] ?? "").replace(/<br\s*\/?>/gi, "\n");  
      }  
