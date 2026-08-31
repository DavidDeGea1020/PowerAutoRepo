# Normalize   
  
  const levelMap: { [key: string]: string } = {  
    "i": "1", "ii": "2", "iii": "3", "iv": "4", "v": "5",  
    "vi": "6", "vii": "7", "viii": "8",  
    "one": "1", "two": "2", "three": "3", "four": "4", "five": "5",  
    "1st": "1", "2nd": "2", "3rd": "3", "4th": "4", "5th": "5",  
    "level1": "1", "level2": "2", "level3": "3"  
  };  
  
  function normalize(text: string): string {  
    const words = text.toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter(w => w.length > 0);  
    return words.map(w => {  
      const expanded = aliases[w] ?? w;  
      return levelMap[expanded] ?? expanded;  
    }).join(" ");  
  }  
