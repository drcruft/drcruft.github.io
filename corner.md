<!-- index2.md -->

# Hello

Hello, you've reached the corner.  :)

<!-- this container holds the data -->
<div id="dataContainer"
     style="display:none; white-space: pre-wrap; margin-top:1rem; background:#f9f9f9; padding:1em; border:1px solid #ddd;">
  {% include_relative data.txt %}
</div>

<!--  hotspot -->
<div id="Hotspot"
     style="
       position: fixed;
       top: 0;
       right: 0;
       width: 20px;
       height: 20px;
       opacity: 0;
       cursor: pointer;
       z-index: 1000;
     "
></div>

<script>
  let revealed = false;
  function revealData() {
    if (revealed) return;
    revealed = true;
    // show the pre-embedded data
    document.getElementById('dataContainer').style.display = 'block';
    // remove the hotspot so it can't be retriggered
    const hs = document.getElementById('Hotspot');
    if (hs) hs.remove();
  }

  // click-hotspot trigger
  document.getElementById('Hotspot')
          .addEventListener('click', revealData);

  // keyboard shortcut trigger: Ctrl+Shift+V
  document.addEventListener('keydown', e => {
    if (!revealed && e.ctrlKey && e.shiftKey && e.key === 'V') {
      revealData();
    }
  });
</script>
