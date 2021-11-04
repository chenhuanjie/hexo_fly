---
layout: frag
---
<canvas id="mainCanvas" onclick="resetTime()"></canvas>
<span id="errorLabel" class="monoLabel"></span>
</div>
<div id="inputBox">
<pre id="stickBox">
<span id="stickContent" class="monoLabel"></span>
</pre>
<textarea id="mainInput" class="monoLabel"></textarea>
</div>
<strong id="warnLabel">
    不推荐直接在这里修改代码, 不仅没有高亮, tab和撤销有时不灵, 搞不好还会弄丢代码, 没必要
</strong>
<style type="text/css">
.monoLabel {
    font-family: monospace;
    font-weight: 700;
    font-size: 14px;
    white-space: pre-wrap;
    word-wrap: break-word;
}
#mainCanvas {
    width: 100%;
    min-height: 200px;
}
#warnLabel {
    display: block;
    margin-top: 16px;
    color: #F75357;
    text-align: center;
}
#errorLabel {
    display: block;
}
#inputBox {
    position: relative;
    line-height: 20px;
    min-height: 100px;
    outline: none;
    padding: 16px;
    box-sizing: border-box;
}
#stickBox {
    display: block;
    padding: 0;
    margin: 0;
    box-sizing: border-box;
    visibility: hidden;
}
#mainInput {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border: none;
    outline: none;
    resize: none;
    padding: 16px;
    box-sizing: border-box;
    background-color: #eee;
    overflow: hidden;
}
</style>
<script>
var textarea = document.getElementById('mainInput');
var container = document.getElementById('inputBox');
var stickSpan = document.getElementById('stickContent');
var vertSrc = '#version 300 es\nin vec4 position; void main(){ gl_Position = position; }';
var compileTimer = 0;
var sampleSrc = `precision highp float;
precision highp int;
uniform float time;
uniform vec2 resolution;
out vec4 fragColor;
void main() {
    vec2 position = gl_FragCoord.xy / resolution;
    float color = 0.0;
    color += sin(position.x * cos(time / 15.0) * 80.0) + cos(position.y * cos(time / 15.0) * 10.0);
    color += sin(position.y * sin(time / 10.0) * 40.0) + cos(position.x * sin(time / 25.0) * 40.0);
    color += sin(position.x * sin(time / 5.0) * 10.0) + sin(position.y * sin(time / 35.0) * 80.0);
    color *= sin(time / 10.0) * 0.5;
    fragColor = vec4(vec3(color, color * 0.5, sin(color + time / 3.0) * 0.75), 1.0);
}`;
function readHashToInput() {
    if (location.hash) {
        var raw = location.hash.replaceAll('%23', '#');
        var txt = decodeURI(raw.substr(1)).trim();
        if (txt) {
            textarea.value = txt.replace('\t', '    ');
            return;
        }
    }
    textarea.value = sampleSrc;
}
function setupInput() {
    readHashToInput();
    stickSpan.textContent = textarea.value;
    textarea.addEventListener('input', function() {
        stickSpan.textContent = textarea.value;
        if (compileTimer)
            clearTimeout(compileTimer);
        compileTimer = setTimeout(() => compileProgram(textarea.value), 1000);
    });
    textarea.addEventListener('keydown', function(event) {
        if (event.keyCode != 9) return;
        document.execCommand('insertText', false, '    ');
        event.preventDefault();
    });
}
setupInput();
// 接下来是canvas相关
var canvas = document.getElementById('mainCanvas');
var gl;
try {
    gl = canvas.getContext('webgl2');
} catch (error) {}
if (!gl) {
    var msg = '当前浏览器不支持WebGL2, 仅显示代码.'
    canvas.style.display = 'none';
    setErrorMsg(msg);
    throw new Error(msg);
}
gl.getExtension('OES_standard_derivatives');
gl.getExtension('GL_EXT_gpu_shader4');
var baseTime = new Date().getTime();
var currentProgram;
var variableCache = {};
function resetTime() {
    baseTime = new Date().getTime();
}
function setErrorMsg(msg) {
    var label = document.getElementById('errorLabel');
    if (msg) {
        msg = msg.substr(0, msg.length - 1);  // remove trailing '\x00'
        label.style.color = 'red';
    }
    else {
        msg = 'Successfully compiled shader program';
        label.style.color = 'lightgreen';
    }
    label.textContent = msg;
}
function createShader(src, type) {
    var shader = gl.createShader(type);
    gl.shaderSource(shader, src);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        setErrorMsg(gl.getShaderInfoLog(shader));
        return null;
    }
    return shader;
}
function compileProgram(src) {
    src = '#version 300 es\n' + src;
    setErrorMsg('');
    var program = gl.createProgram();
    var vert = createShader(vertSrc, gl.VERTEX_SHADER);
    var frag = createShader(src, gl.FRAGMENT_SHADER);
    if (vert == null || frag == null) return;
    gl.attachShader(program, vert);
    gl.attachShader(program, frag);
    gl.deleteShader(vert);
    gl.deleteShader(frag);
    gl.linkProgram(program);
    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
        setErrorMsg(gl.getProgramInfoLog(program));
        return;
    }
    console.log('Successfully compiled shader program');
    if (currentProgram)
        gl.deleteProgram(currentProgram);
    currentProgram = program;
    gl.useProgram(program);
    variableCache.position = gl.getAttribLocation(program, 'position');
    gl.enableVertexAttribArray(variableCache.position);
    variableCache.resolution = gl.getUniformLocation(program, 'resolution');
    variableCache.time = gl.getUniformLocation(program, 'time');
}
compileProgram(textarea.value);
function getPositionBuffer() {
    var vertices = new Float32Array([-1,-1, 1,-1, -1,1, 1,-1, 1,1, -1,1]);
    var vbuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, vbuffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);
    return vbuffer;
}
var buffer = getPositionBuffer();
function resizeCanvas() {
    var displayWidth = canvas.clientWidth;
    var displayHeight = canvas.clientHeight;
    var expectation = parseInt(displayWidth / 16 * 9);
    if (canvas.width != displayWidth
            || canvas.height != displayHeight
            || displayHeight != expectation) {
        canvas.style.height = expectation + 'px';
        var width = canvas.clientWidth;
        var height = canvas.clientHeight;
        canvas.width = width;
        canvas.height = height;
        gl.viewport(0, 0, width, height);
    }
}
function drawScene() {
    requestAnimationFrame(drawScene);
    resizeCanvas();
    if (!currentProgram)
        return;
    gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);
    gl.useProgram(currentProgram);
    gl.uniform2f(variableCache.resolution, canvas.width, canvas.height);
    var diffTime = (new Date().getTime() - baseTime) / 1000;
    gl.uniform1f(variableCache.time, diffTime);
    gl.vertexAttribPointer(variableCache.position, 2, gl.FLOAT, false, 0, 0);
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bindFramebuffer(gl.FRAMEBUFFER, null);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
}
window.addEventListener('load', drawScene);
</script>
