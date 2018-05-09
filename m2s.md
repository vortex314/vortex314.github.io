---


---

<h1 id="mqtt-to-serial-command-line">mqtt to serial command line</h1>
<p>this tool works together with serial2mqtt to read and display the serial output, but also to send a binary file to be flashed.</p>
<p>Different modes of operation :</p>
<ul>
<li>
<p>manual, waits for a key press § to send next version of binary file to dst/DEVICE-SERIAL/serial2mqtt/flash</p>
</li>
<li>
<p>all data received on src/DEVICE-SERIAL/serial2mqtt/log is just displayed on stdout.</p>
</li>
<li>
<p>all mqtt published by device can also be monitored by supplying -m options. The logical device is found on src/DEVICE-SERIAL/serial2mqtt/device where the logical DEVICE will appear.</p>
</li>
</ul>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

