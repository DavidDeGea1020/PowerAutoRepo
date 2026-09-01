# Skill  
  
---  
name: jd-title-lookup  
description: Use this skill whenever a user wants to view, look up, read, check, or update a job description, or mentions a job title, job code, or position. Governs how to find the correct job description using the Match Job Title tool and how to handle ambiguous or unmatched titles.  
---  
  
# Job Description Title Lookup  
  
## When to apply  
Apply this skill any time the user references a job position for viewing or updating — even vaguely ("that compliance job", "the analyst role").  
  
## Core rule  
ALWAYS call the Match Job Title tool with the user's stated title before displaying, discussing, or updating any job description. Never answer from memory or guess which job the user means. Never invent a job title that was not returned by the tool.  
  
## Interpreting the Candidates result  
The tool returns up to 5 candidates as JSON: id, title, jobCode, score (0-100).  
  
Apply these rules in order:  
  
1. **Confident match**: top score is 85 or higher AND at least 15 points above the second candidate.  
   → Present that single title and ask the user to confirm before doing anything else.  
   Example: "I found **BSA Monitoring Analyst**. Is this the position you'd like to view?"  
  
2. **Close contenders**: two or more candidates within 10 points of the top score.  
   → Present ONLY those close candidates (not all 5) as a numbered list and ask which one the user means. If they differ only by level (I/II/III), ask which level.  
  
3. **Weak results**: top score below 50.  
   → Tell the user nothing matched well. Ask them to try the fuller official title. You may offer the top 1-2 candidates as possibilities, clearly framed as long shots.  
  
4. **Everything else** (top score 50-84 without close contenders):  
   → Present the top candidate as a tentative match and ask the user to confirm, offering the next 1-2 as alternatives.  
  
## Display rules  
- Never show numeric scores or internal ids to the user.  
- Job codes may be shown only if the user asks or provides one.  
- Never proceed to display or modify a job description without an explicit user confirmation of the matched title.  
  
## After confirmation  
- If the user confirms and wants to VIEW: retrieve the full job description details for the confirmed job code and present them.  
- If the user confirms and wants to UPDATE: follow the update process (updates require HR approval; do not modify data directly).  
- If the user rejects the match: ask for more detail and call the Match Job Title tool again with their refined input.  
