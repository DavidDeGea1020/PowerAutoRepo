# Update ingestion  
  
    const rawTitle = obj["Job Title"] ?? "";  
    const dashAt = rawTitle.indexOf("-");  
    if (dashAt > 0 && /^[A-Za-z]{1,6}\d+\s*$/.test(rawTitle.substring(0, dashAt).trim())) {  
      obj["Job Code"] = rawTitle.substring(0, dashAt).trim();  
      obj["Job Title"] = rawTitle.substring(dashAt + 1).trim();  
    }  
