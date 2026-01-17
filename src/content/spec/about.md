# About
I post CTF Writeups and things I learn so I don't have to Google them again. If you identify any error, feel free to contact me via X. I'm still learning, so it would be great to know. Also, I might post about other non-computer related things like travels, food...

<div style="overflow: hidden; white-space: nowrap;">
  <div id="monkey-track" style="display: inline-block; animation: monkey-scroll 8s linear infinite; animation-delay: 2s; padding-left: 100%; font-size: 3rem; cursor: pointer; white-space: pre;"><span style="font-size: 1.5rem; vertical-align: middle;">🍌</span>                             🐒</div>
</div>
<style>
  @keyframes monkey-scroll {
    from { transform: translateX(0); }
    to { transform: translateX(-100%); }
  }
  #monkey-track:hover {
    animation-play-state: paused;
  }
  #monkey-track {
    padding-top: 2rem;
  }
</style>
<script>
  (function() {
    try {
      const initMonkey = () => {
        const track = document.getElementById('monkey-track');
        if (!track) return;
        let gap = 40;
        let state = 0;
        let heli = false;
        const bananaHTML = '<span style="font-size: 1.5rem; vertical-align: middle;">🍌</span>';
        const runCutscene = () => {
           track.style.animation = 'none';
           track.style.paddingLeft = '0';
           track.style.textAlign = 'center';
           track.style.width = '100%';
           track.style.height = '2.5em';
           // Cheetah
           track.innerHTML = bananaHTML + " <span style='display:inline-block; transform: scaleX(-1);'>🐆</span> 🐒";
           setTimeout(() => {
              // Knife
              track.innerHTML = bananaHTML + " <span style='display:inline-block; transform: scaleX(-1);'>🐆</span> <span style='display:inline-block; font-size:1.5rem; vertical-align:middle; transform: scaleX(-1);'>🔪</span> 🐒";
              setTimeout(() => {
                  // Tombstone
                  track.innerHTML = bananaHTML + " 🪦 🐒";
                  setTimeout(() => {
                 // Swap (Monkey takes place)
                 track.innerHTML = bananaHTML + " 🐒";
                 setTimeout(() => {
                    // Police (Monkey Center, Police Far Right)
                    track.style.display = 'block';
                    track.style.position = 'relative';
                    track.innerHTML = "<span style='position:absolute;left:50%;transform:translateX(-50%);'>🐒</span><span style='position:absolute;right:0;'>🚓</span>";
                    setTimeout(() => {
                       // Oops
                       track.innerHTML = "<span style='position:absolute;left:50%;transform:translateX(-50%);'>🙊</span><span style='position:absolute;right:0;'>🚓</span>";
                       setTimeout(() => {
                          // Run Start
                          state = 6;
                          gap = 30;
                          heli = false;
                          track.innerHTML = "🐒💨" + " ".repeat(gap) + "🚓";
                          track.style.display = 'inline-block';
                          track.style.position = '';
                          track.style.paddingLeft = '50%';
                          track.style.textAlign = '';
                          track.style.width = '';
                          track.style.height = '';
                          track.style.animation = 'monkey-scroll 8s linear infinite';
                          setTimeout(() => {
                             if (state === 6) {
                                heli = true;
                                track.style.animation = 'none';
                                track.offsetHeight; 
                                track.style.animation = 'monkey-scroll 8s linear infinite';
                                track.innerHTML = "🐒💨" + " ".repeat(gap) + "<span style='position:relative;display:inline-block;'>🚓<span style='position:absolute;top:-1.2em;right:-0.5em;'>🚁</span></span>";
                             }
                          }, 5000);
                          setTimeout(() => {
                             state = 8;
                             track.style.animation = 'none';
                             track.style.paddingLeft = '0';
                             track.style.textAlign = 'center';
                             track.style.width = '100%';
                             track.style.fontSize = '1.5rem';
                             track.innerHTML = "✨ To Be Continued... 🐒 ✨";
                          }, 15000);
                       }, 2000);
                    }, 2000);
                 }, 2000);
              }, 2000);
              }, 1500);
           }, 2000);
        };
        const update = () => {
          if (state === 0) { // Chase
            if (gap > 0) {
              gap -= 25;
              if (gap <= 0) {
                 runCutscene();
              } else {
                 track.innerHTML = bananaHTML + " ".repeat(gap) + "🐒";
              }
            } else {
              gap = 40;
              track.style.paddingLeft = '100%';
              track.innerHTML = bananaHTML + " ".repeat(gap) + "🐒";
            }
          } else if (state === 6) { // Police Chase
            gap += 5;
            if (gap > 50) {
               state = 0;
               gap = 40;
               track.style.paddingLeft = '100%';
               track.innerHTML = bananaHTML + " ".repeat(gap) + "🐒";
            } else {
               track.innerHTML = "🐒💨" + " ".repeat(gap) + (heli ? "<span style='position:relative;display:inline-block;'>🚓<span style='position:absolute;top:-1.2em;right:-0.5em;'>🚁</span></span>" : "🚓");
            }
          }
        };
        track.addEventListener('animationiteration', update);
        track.addEventListener('webkitAnimationIteration', update);
      };
      if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initMonkey);
      } else {
        initMonkey();
      }
    } catch (e) { console.error(e); }
  })();
</script>
