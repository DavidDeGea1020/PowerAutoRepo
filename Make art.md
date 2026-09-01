# Make art  
  
    const rawTitle = (obj["Job Title"] ?? "").trim();  
    const code = (obj["Job Code"] ?? "").trim();  
    if (code.length > 0 && rawTitle.toUpperCase().startsWith(code.toUpperCase())) {  
      obj["Job Title"] = rawTitle.substring(code.length).replace(/^\s*-\s*/, "").trim();  
    }  
