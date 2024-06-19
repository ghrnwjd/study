## Flask, IoT 샘플 코드

### [index.html]
```js
<script src=
"https://cdnjs.cloudflare.com/ajax/libs/socket.io/2.1.1/socket.io.js">
</script>
```

### [iot_with_flask.js]
```js
var socket = io.connect('http://'+ document.domain +':'+ location.port);
```

- documnet.domain은 라즈베리파이IP를 가져온다. 

ex) https://www.naver.com/abcd/efgh -> www.naver.com
```js
socket.on('connect',function(){
     console.log('Websocket connected!');
});
```
- 클라이언트 사이드 스크립트가 서버와 연결되면 웹 소켓이 성공적으로 연결됬다고 콘솔 입력
 

### [index.html]
```html
<button onclick="measurement_start()">Start</button>
```

- html에서 버튼을 클릭하면 자바스크립트의 measurement_start 함수가 실행된다.
 
### [iot_with_flask.js]
```py
function measurement_start(){
  console.log('start...');
  measurementCnt =document.getElementById("measurementCnt");
  socket.emit('measurement_start',
    {iterations:parseInt(measurementCnt.value)});
}
```

- 사용자가 html 에서 입력한 measurementCnt의 value를 소켓을 통해 measurement_start라는 키로 보낸다.
- 여기서 measurement_start는 light를 측정하는 횟수를 의미한다.

### [app.py]
```py
@socketio.on('measurement_start')
def on_create(data):
  print("measurement_start")
  it=data['iterations']
  for i in range(it):
    wp.digitalWrite(ADC_CS,1)
    buf = bytearray(3)
    buf[0] = 0x06 |((ADC_CH_LIGHT &0x04)>>2)
    buf[1] = ((ADC_CH_LIGHT &0x03)<<6)
    buf[2] = 0x00
    wp.digitalWrite(ADC_CS,0)
    ret_len,buf = wp.wiringPiSPIDataRW(SPI_CH,bytes(buf))
    buf = bytearray(buf)
    buf[1] = 0x0F &buf[1]
    #value=(buf[1] << 8) | buf[2]
    light_val = int.from_bytes(bytes(buf),byteorder='big')
    wp.digitalWrite(ADC_CS,1)
    print("light value=",light_val)
    socketio.sleep(.2)
    emit('msg_from_server', {'lightVal':light_val} )
```

- 서버는 클라이언트로부터 measurement_start를 소켓을 통해 전달 받으면 on_create 함수를 실행시킨다.
- wiringPi 를 이용하여 센서를 동작시키고, 측정된 light value를 클라이언트에 msg_from_server라는 키로 전달한다.

```js
[iot_with_flask.js]
socket.on('msg_from_server',function(msg){
  console.log(msg);
  var logArea = document.getElementById("log");
  if(msg == null) {
    logArea.textContent = '';
  }
  else {
    logArea.textContent = 'light value='
    + msg.lightVal +'\n'+ logArea.textContent;     
  }
});
```

- 서버에서 전달한 데이터가 클라이언트 콜백함수 인자로 담겨온다.
- msg.lightVal을 통해 서버에서 보낸 데이터를 출력한다.

### [app.py]
```py
@app.route("/led1/<led_state>")
def control_led_action(led_state):
  print("control_led_action")
    
  if led_state =="true":
    print("action==true")
    wp.digitalWrite(LED_PIN,1)
    ledS = "ON"
  else:
    print("action==false")
    wp.digitalWrite(LED_PIN,0)
    ledS = "OFF"
      
  now =datetime.datetime.now()
  timeString =now.strftime("%Y-%m-%d %H:%M:%S")
  templateData ={
    'time':timeString ,
    'ledS':ledS
  }                  
  return render_template('ajax_led_response.html', **templateData)
```

- 사용자가 http://라즈베리파이IP/led1/입력값으로 GET 요청을 보내면 사용자의 입력값이 control_led_action 함수의 인자로 들어오며 해당 함수가 실행된다.
- 현재 시간을 datetime.datetime.now()로 불러오고 해당 객체의 strftime 함수를 통해 날짜 포맷형식을 정해줄 수 있다.
- %Y/%m/%d %H:%M:%S -> 2024/06/20 00:09:12
- 월, 일은 소문자인 것에 주의할 것