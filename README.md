# spideruino
A remote-controlled ESP8266-based hexapod.

## Demo Time!
[![Youtube Video](https://img.youtube.com/vi/oVFrtwVP7xY/0.jpg)](https://www.youtube.com/watch?v=oVFrtwVP7xY)

### Motivation
Spurred by the combination of [controlling a hexapod via Haskell code](http://blog.sigfpe.com/2011/02/build-yourself-bluetooth-controlled-six.html), I planned to build the first ESP8266-based hexapod running solely on [Purescript](http://www.purescript.org/). You might ask why? Good question! A comment on [Reddit](https://www.reddit.com/r/esp8266/comments/3xein6/alternative_way_to_develop_esp8266based_project/cy7mu9z) regarding alternative ways to develop ESP8266-based projects set the "final trigger" :)

_[...] But if you want to be really unique, use Evil mode on Emacs and cross compile PureScript code to Javascript and run that JavaScript on the ESP8266. You will very likely be the only person on the whole planet using that combination._

\*cough\* After 2 days trying to glue NodeJS respectively [Espruino](http://www.espruino.com/)[^1] APIs via Purescripts [Foreign Function Interface](http://www.purescript.org/learn/ffi/), I finally pivot to old-school C++ to finish the job, and loosing thereby all of the $wag.

[^1]: Espruino allows to use JavaScript directly on a ESP chip.

At this point you might be asking *what the heck is a [ESP8266](https://en.wikipedia.org/wiki/ESP8266)?* I'm glad that you asked. Originally it was/is used as a WiFi shield for [Arduino computers](https://www.arduino.cc/). But the truth of the matter is that the ESP is already powerful enough for building diverse DIY applications. Just look at the vast amount of projects based on it: [https://hackaday.com/tag/esp8266/](https://hackaday.com/tag/esp8266/).

### ELI5: How did you build it?

You'll need:

* 1 x ESP8266 201 chip
* 3 x Micro Servos (3,7g)
* 2 x Lipo batteries (240 mAh at 3,7 V)
* 1 x 330 Ohm resistor
* 1 x 1k Ohm resistor
* 1 x 100 nF capacitor
* 1 x soldering iron
* 1 x hot glue gun
* some wires

The approx. total cost will be around 10 - 20€ depending on your existing equipment.

#### Circuit Diagram & Breadboard Configuration
The spideruino [Github repository](https://github.com/qabbasi/spideruino) also contains [Fritzing diagrams](http://fritzing.org/home/) displaying the actual circuit diagram and breadboard configuration. If you want to use them, install the following module beforehand: [Fritzing ESP8266-201 module](https://github.com/ydonnelly/ESP8266_fritzing/blob/master/ESP8266-201%20WiFi%20Module.fzpz).
The original ESP pin out can be found [here](https://www.mikrocontroller.net/attachment/307864/esp8266_esp_201_module_pinout_diagram_cheat_sheet_by_adlerweb-d9iwmqp.jpg).
If you have trouble to follow along, then have a look at the building instructions at a similar project: [pololu](https://pololu.com/docs/0J42/all).

Notice: To upload your code from the Arduino IDE to the ESP, you have to ground the pin 0 in order to boot in _serial programming_ mode.
[Here](https://github.com/esp8266/Arduino) you can learn how to import the ESP board into the Arduino IDE.

### spideruino – Quo Vadis?
Before examining the code, let's figure out how spideruino moves at all.

![s](https://a.pololu-files.com/picture/0J2123.1200.jpg?c47bf4a217b05b97661d7efb1f168f28)

Right legs are front, left legs are back and middle leg (left) is touching the ground.

![s](https://a.pololu-files.com/picture/0J2124.1200.jpg?dfd0a2ed1171f439c8611155e0d1365c)

Right legs are back, left legs are front and middle leg (left) is touching the ground.

![s](https://a.pololu-files.com/picture/0J2125.1200.jpg?bc0742dee89f1c0a71a8340e4f1baddd)

Right legs are back, left legs are front and middle leg (right) is touching the ground

![s](https://a.pololu-files.com/picture/0J2122.1200.jpg?4401a4d78dcd5bbd45be504f095e6fa2)

Right legs are front, left legs are back and middle leg (right) is touching the ground.

*In a nutshell:* The frames are displaying the 'forward' movement pattern of the spideruino gait. As you probably figured out, the main idea is to move diagonally while shifting/alternating the weight around the center line.

This lead to the following design:

![Movement pattern](http://i.imgur.com/Zih4qwql.png)

The idea here is to map the shown pattern to ultimately servo positions. The servo positions again are divided into frames, which will be played sequentially depending on the moving direction.

### Show me the code

```
int forward[4][3] = {{70, 95, 60}, {110, 93, 100}, {110, 83, 100}, {80, 85, 70}};
int backward[4][3] = {{70, 95, 60}, {80, 85, 70}, {110, 83, 100}, {110, 93, 100}};
int left[4][3] = {{115, 95, 60}, {100, 98, 100}, {100, 80, 100}, {115, 90, 60}};
int right[4][3] = {{115, 82, 70}, {90, 82, 110}, {90, 95, 110}, {116, 97, 75}};

void setup() {
  Serial.begin(115200);

  servoLeft.attach(13);
  servoMid.attach(12);
  servoRight.attach(14);

  Serial.println();
  Serial.print("Configuring access point...");
  
  WiFi.softAP(ssid);

  IPAddress myIP = WiFi.softAPIP();
  Serial.print("AP IP address: ");
  Serial.println(myIP);
	
  server.on("/", handleRoot);
  server.on("/position", handlePosition);
  server.on("/forward", handleForward);
  server.on("/backward", handleBackward);
  server.on("/left", handleLeft);
  server.on("/right", handleRight);
  server.on("/reset", handleReset);
  server.begin();
  Serial.println("HTTP server started. Waiting for client requests...");
}

void handleForward() {
  handleMovement(forward);
}

void handleMovement(int coords[4][3]) {
  resetPosition();

  for (int i = 0; i <= 3; i++) {
    int left = coords[i][0];
    int mid = coords[i][1];
    int right = coords[i][2];

    run(left, mid, right);

    delay(150);
  }
  responseOK();
}

void run(int left, int mid, int right) {
  servoLeft.write(left);
  servoMid.write(mid);
  servoRight.write(right);
}
```

*Note:* For better readability I trimmed the listing to the relevant parts.

As you can see, `setup()` is creating an WiFi access point and binding a HTTP web server on port 80. You might be guessing what `loop()` is doing... it's basically a callback function to handle incoming HTTP client requests. The rest of the code should be trivial to understand: when for example `/forward` is called then the well tuned frames (i.e. arrays of servo positions) will be simply looped over in order to perform the corresponding movement.

### Finally...

With just some lines of code you can also build a remote-controlled hexapod too! If so, I hope that this post will be helpful to you.
