---
layout: post
title: "Bla"
---

Bla bla bla bla.

<script type="text/javascript">
function start_emu() {
    if (!document.getElementById('emu')) {
        let emubox = document.getElementById('emubox')
        emubox.innerHTML = 'Close Me<br>'
        emubox.onclick = stop_emu
        let emu = document.createElement('iframe')
        emubox.appendChild(emu)
        emu.id = 'emu'
        emu.src = 'https://floooh.github.io/tiny8bit/cpc.html?file=cpc/backtro.sna'
        emu.style = 'border:0;'
        emu.width = '512px'
        emu.height = '384px'
    }
}
function stop_emu() {
    let emubox = document.getElementById('emubox')
    emubox.innerHTML = 'Click Me!'
    emubox.onclick = start_emu
}
</script>
<a style="cursor:pointer" id="emubox" onclick="start_emu();">Click Me!</a>

Blub...
