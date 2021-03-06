# Oboo Cards

Card System for Oboo Smart Clock

# Docker Quickstart

Steps to configure Docker:
* Build the Oboo Cards Docker Image
	* `docker build -t oboo-cards .``
* Start an Oboo Cards container
	* `docker run -it oboo-cards /bin/bash`
	* Allows you to compile and run the program in the container
* Start an Oboo Cards container with the host computer’s directory mounted as a volume:
	* `docker run -it -v $(pwd):/usr/src/myapp oboo-cards /bin/bash`
	* **Recommended command**
	* Allows you to:
		* Make changes in host machine environment
		* Compile and run the program in the container

# Oboo Runtime

Lightweight Javascript ES5 runtime

## Runtime Intro

C code:

* All located in `oboo-runtime` directory
* `ort.c`
	* Holds `main()` function
* `runtime.c`
	* Defines all C functions that can be used in Javascript
	* `initRuntime()` function registers these C functions for use in Javascript
	* Embedded Javascript engine
		* Using `duktape` library
		* Really good documentation online: https://duktape.org/
* `messageQueue.c`
	* Holds all MQTT functions
	* Based on `mosquitto` library
* `serialPort.c`
	* Holds all serial port communication functions
	* Based on `termios` library

Compiled `ort` binary should be run as follows:

```
./ort <path to JS file>
```

## Quickstart

* Make sure mosquitto broker is running
	* `service mosquitto start`
* Compile the code
	* `make`
* Run the test program
	* `make test`


## Available Functions

Listing of functions available in the Javascript runtime

### System Functions

#### writeFile

Description:

Write to a file on the filesystem.

Syntax:

```
writeFile(filePath, fileContents, callback(err) )
```

Parameters:

* `filePath`
	* Path to target file
	* If a file does not exist, it will be created. If it exists, it will be overwritten
* `fileContents`
	* Text to write to the target file
* `callback(err)`
	* Callback function
	* `err` parameter will be null if write is successful

#### readFile

Description:

Write to a file on the filesystem.

Syntax:

```
readFile(filePath, encoding, callback(err, data) )
```

Parameters:

* `filePath`
	* Path to target file
	* If a file does not exist, it will be created. If it exists, it will be overwritten
* `encoding`
	* Not used at the moment
* `callback(err, data)`
	* Callback function
	* `err` parameter will be null if read is successful
	* `data` will hold contets of the file

#### setGpio

Description:

Use `gpioctl` utility to set a GPIO pin to a specified output value

Syntax:

```
setGpio(gpioNum, value)
```

Parameters:

* `gpioNum`
	* Number of GPIO to set output
* `value`
	* Value to set GPIO
	* Should be `0` (low) or `1` (high)

#### playMP3

Description:

Use `mpg123` utility to play specified MP3 music file

Syntax:

```
playMP3(filePath)
```

Parameters:

* `filePath`
	* Path to target audio file
	* Supported formats: `mp3`

#### system

Description:

Make any system call **NOTE: this can be dangerous!**
This is a blocking call.

Syntax:

```
system(command)
```

Parameters:

* `command`
	* Command line string to execute

#### popen

Description:

Makes any system call and returns the output of the command (stdout and stderr) as a string **NOTE: this can be dangerous!** 
This is a blocking call.

Syntax:

```
val = popen(command)
```

Parameters:

* `command`
	* Command line string to execute

Returns:

* A string that holds the stdout and stderr output of the executed command

#### launchProcess

Description:

Allows you to run system call and returns PID of particular process
This is non blocking call.

Syntax:

```
val = launchProcess(command)
```
**NOTE: Use this call with complete set of parametars like (mpg123 --loop -1 somefile)**
Returns:

* A PID of new process or 0

#### killProcess

Description:

Allows you to kill process with PID
This is non blocking call.

Syntax:

```
val = killProcess(PID)
```
Returns:
1 on sucess 0 on failure

#### checkProcess

Description:

Allows you to check if process with PID is running

Syntax:

```
val = checkProcess(PID)
```
Returns:
1 if process is running 0 if not

### Message Queue Functions

#### connect

Description:

Initiate connection to message queue broker as a client

Syntax:

```
connect(host, port, [clientId], [callback()], [enableTls], [willTopic], [willPayload])
```

Parameters:

* `host`
	* Address of message queue broker
* `port`
	* Port of message queue broker
* `clientId`
	* Client ID to take when connected to message queue broker
	* *Optional*
		* If left blank, this client ID will be autogenerated
* `callback()`
	* *Optional*
	* Callback function
		* TODO: this function should give some indication whether it was successful or not
* `enableTls`
	* *Optional*
	* Boolean to enable TLS secured connection
		* TODO: needs more work to be programmatic
* `willTopic`
	* *Optional*
	* Can specify an MQTT will for the client
	* This sets the **topic** for the will, must be used in conjunction with `willPayload`
* `willPayload`
	* *Optional*
	* Can specify an MQTT will for the client
	* This sets the **payload** for the will, must be used in conjunction with `willTopic`

#### subscribe

Description:

Subscribe to a topic. Must be run after `connect` function.

Syntax:

```
subscribe(topic, [callback()], [enableTls])
```

Parameters:

* `topic`
	* String identifying the topic to which to subscribe
* `callback()`
	* *Optional*
	* Callback function
		* TODO: this function should give some indication whether it was successful or not
* `enableTls`
	* *Optional*
	* Boolean to enable using TLS secured connection
		* TODO: this needs to be overhauled

#### subscribe

Description:

Subscribe to a topic. Must be run after `connect` function.

Syntax:

```
publish(topic, payload, [enableTls])
```

Parameters:

Parameters:

* `topic`
	* String identifying the topic to which to publish paylod
* `payload`
	* String payload to publish
* `enableTls`
	* *Optional*
	* Boolean to enable using TLS secured connection
		* TODO: this needs to be overhauled


## Exiting a Runtime Program

Currently, it's only possible to exit an Oboo Runtime Javascript program in the JS `loop()` function by returning `-1`.

Example:

```
function loop() {
	if (globalVar === true) {
		// exit this program
		return -1;
	} else {
		print('Still running!');
	}
}
```

## Hardware Interaction

All received MQTT messages call the `onRecvMessage()` function that's defined in `oboo-runtime/js/card-lib.js`.

This function will then call `onInput()` or `onMessage()` functions that **must** be defined in each Oboo Runtime Javascript program to function properly.

### MQTT Messages

Regular MQTT messages will be passed as follows:

```
onMessage({
	topic: messageTopic,
	payload: messagePayload
})
```

### Hardware Input

User input from the touch pad or gesture sensor will be passed to the Oboo Runtime Javascript program through the `onInput()`.

#### Touch Input

The Hardware supports multi-touch (detection of simultaneous presses of multiple buttons) and tracks the duration of the button press. The message will be sent out when the button is released.

##### Single Touches

For single touch button presses, it will be invoked as follows:

```
onInput({
	source: 'button',
	payload: {
		id: buttonId, // defines buttons left to right as 0 to 3
		action: 'press',
		duration: 123, 	// time in milliseconds button was pressed
		multitouch: false
	}
});
```

##### Multi-Touch

For multi-touch, the function will be invoked as follows:

```
onInput({
	source: 'button',
	payload: {
		id: buttonByte, // see description below
		action: 'press',
		duration: 123, 	// time in milliseconds button was pressed
		multitouch: true
	}
});
```

The `buttonByte` is an integer, with each bit representing a different button:

| Bit 3                     | Bit 2    | Bit 1    | Bit 0                    |
|---------------------------|----------|----------|--------------------------|
| Button 3 (Farthest Right) | Button 2 | Button 1 | Button 0 (Farthest Left) |

So a value of `14` or `0xe` in hex would be `0b1110`, meaning buttons 1-3 were pressed.

#### Gesture Input

For gesture input, it will be invoked as follows:

```
onInput({
	source: 'gesture',
	payload: {
		direction: ['left'|'right'|'up'|'down'|'none']
	}
});
```
