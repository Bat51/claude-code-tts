Title: Fix: hooks silently no-op on Linux (tail -r → tac) + add KOKORO_LANG for non-English voices                                                                  
## Summary                                                                                                                                                                          
                                                                                                                                                                                
Two related changes for non-macOS / non-English users.                                                                                                                              
                                                                                                                                                                                
### 1. Bug fix — hooks never read the transcript on Linux                                                                                                                           
                                                                                                                                                                                
`tts-stop-hook.sh` and `tts-pretooluse-hook.sh` use `tail -r` to walk the                                                                                                           
JSONL transcript in reverse. `tail -r` is a BSD/macOS extension and does                                                                                                            
not exist in GNU coreutils. On Linux, the command errors out immediately,                              
the read loop never iterates, and `claude_response` stays empty — so the                                                                                                            
hook silently produces no audio for **every** Linux user.                                                                                                                           
                                                                                                                                                                                
Symptom in the log:                                                                                                                                                                 
[...] Extracted response:                                                                                                                                                           
[...] No response found                                                                                                                                                             
on every Stop event, even though `last_assistant_message` in the hook                                                                                                               
input contains the full text.                                                                                                                                                       
                                                                                                                                                                                
Fix: try `tac` first (GNU), fall back to `tail -r` (BSD). Both platforms                                                                                                            
now work.                                                                                                                                                                           
                                                                                                                                                                                
```bash                                                                                                                                                                             
done < <(tac "$transcript_path" 2>/dev/null || tail -r "$transcript_path")                                                                                                        
                                                                                                                                                                              
2. Feature — KOKORO_LANG env var                                                                                                                                                    
                                                                                                                                                                                
kokoro-tts defaults to en-us. With a non-English voice (e.g.                                                                                                                        
ff_siwis for French, or any jf_* / zf_* / ef_*…), pronunciation                                                                                                                     
remains anglicized unless --lang is passed. Users currently have no                                                                                                                 
way to set this from settings.json.                                                                                                                                                 
                                                                                                                                                                                
Add a KOKORO_LANG env var (default en-us) read the same way as                                                                                                                      
KOKORO_VOICE, and forward it to kokoro-tts --lang.                                                                                                                                  
                                                                                                                                                                                
Example ~/.claude/settings.json:                                                                                                                                                    
"env": {                                                                                                                                                                            
"KOKORO_VOICE": "ff_siwis",                                                                                                                                                       
"KOKORO_LANG": "fr-fr"                                                                                                                                                          
}                                                                                                                                                                                 
                                                                                                                                                                                
Supported codes (from kokoro-tts --help-languages):
cmn, en-gb, en-us, fr-fr, it, ja.                                                                                                                                                   
                                                                                                                                                                                
Test plan                                                                                                                                                                           
                                                                                                                                                                                
- Linux (Ubuntu, GNU coreutils): hook now extracts the response and plays audio. Verified via /tmp/kokoro-hook.log.                                                                 
- Verified tac falls through cleanly when present, tail -r fallback path is unreachable on Linux.                                                                                 
- macOS: not retested by me — the tac 2>/dev/null || tail -r pattern preserves macOS behavior since tac does not exist there, the || triggers, and tail -r runs as before.          
- French TTS with KOKORO_VOICE=ff_siwis + KOKORO_LANG=fr-fr: pronunciation is correctly French (vs anglicized French without the flag).                                             
                                                                                                                                                                                
Notes                                                                                                                                                                               
                                                                                                                                                                                
The two changes are split into two commits so you can cherry-pick the                                                                                                               
bug fix alone if you prefer to land the feature separately.   
