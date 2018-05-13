# video-text-relay
Simple Gstreamer based video stream relay which can overlay text of various sorts. Text can be set by tcp clients via JSON RPC calls, allowing simply coded daemons to dynamically change text superimposed on video streams.
![Overlay Demo in VLC](https://raw.githubusercontent.com/on-three/video-text-relay/master/img/vlcsnap-2014-06-18-18h00m31s153.png)

# Overview
This code builds a single executable, `video-text-relay`, which does the following:
* Connects to an HTTP served video stream (of various containers/audio+video encodings)
* Renders text onto the decoded video stream (with possible text markup)
* Reencodes the video to a standard MPEGTS(h264/mp3) video with text overlays
* Relays the video stream to a local TCP server currently at 127.0.0.1:10000 (configurable)
* Text can be added/removed via JSON RPC calls currently on 127.0.0.1:8080 (configurable)
* An instance of VLC or some other application can relay the TCP stream via HTTP to any client.

![application diagram](https://raw.githubusercontent.com/on-three/video-text-relay/master/img/video_text_relay.png)

None of this is very involved, and it can be considered pretty 'typical' GStreamer code.

# building
I've tried to make it as simple to compile as possible, but this still relies upon having development packages for Gtk and GStreamer being available. I've only built it in LinuxMint16, where the following command _should_ cover all the Package Dependencies:

```
sudo apt-get install g++ libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libcairo2-dev libglib2.0-dev libgstreamer-plugins-good1.0-dev libpangocairo-1.0-0 libtclap-dev libjsoncpp-dev libboost-regex-dev
```

If those packages are available, you should be able to simply download the repository, move to the src directory and build.

```
git clone https://github.com/on-three/video-text-relay
cd video-text-relay/src
make
```

Haven't tried to build on Windows yet. In theory it ought to build, but I don't think I'll take the time to do so.


## running video-text-relay
Run as below. The stream URL is  currently the only argument. TCP Server port is still hard coded to 10000.
```
./video-text-relay "http://<stream URL>:<PORT>/;stream.nsv"
[MSG]	Received new pad 'src_0' from 'demux':
[OK]	Pipeline of type 'video/x-raw' is now online.
[MSG]	Received new pad 'src_1' from 'demux':
[OK]	Pipeline of type 'audio/x-raw' is now online.

```
You can then connect VLC (and other media players?) to 'TCP://127.0.0.1:10000' and you should pick up the relay (with as yet no text!).

## Displaying Text on the Relayed Video Stream
To add text overlays to the relayed video stream, 'video-text-realay' runs a simple http server that can pick up JSON RPC messages on (currently) port 8080. This allows JSON messages to control the text overlaid on the relayed stream. The required code for clients is fairly simple, and cound conceivably be coded in any language, but I'm currently using python.

Python scripts to set text on a running relay are available in the /python directory.

### scrolling_msg
This python script adds scrolling msg text to the video. Simple command line help is available:
```
./python/scrolling_msg -h
usage: scrolling_msg [-h] [-m MESSAGE] [-u URL] [-f FONT] [-t SCROLL_TIME]
                     [-l LOOPS] [-y YPOS] [-d]
                     message_name

Add or remove a scrolling message on a video stream.

positional arguments:
  message_name          Friendly name to remove/change this message later.

optional arguments:
  -h, --help            show this help message and exit
  -m MESSAGE, --message MESSAGE
                        Text to display on video stream scrolling msg
  -u URL, --url URL     URL of json RPC server to invoke commands on in form
                        IP:PORT.
  -f FONT, --font FONT  Pangocairo font family and size.
  -t SCROLL_TIME, --scroll_time SCROLL_TIME
                        Time in seconds to scroll text across screen.
  -l LOOPS, --loops LOOPS
                        Number of times (loops) to scroll the text. Value 0 is
                        show forever.
  -y YPOS, --ypos YPOS  Vertical y pos of scrolling text in pixels.
  -d, --delete          Delte scrolling message via the provided message name.

```
An example of superimposed video text is shown below.

![Overlay Demo in VLC](https://raw.githubusercontent.com/on-three/video-text-relay/master/img/Screenshot%20from%202014-04-28%2018:49:00.png)

### static_msg
This python script can apply staic (non moving) text to a specific location (x,y in pixels) over the relay stream. Simple help is avilable as below:
```
./python/static_msg -h
usage: static_msg [-h] [-m MESSAGE] [-u URL] [-f FONT] [-x XPOS] [-y YPOS]
                  [-d]
                  message_name

Add or remove a scrolling message on a video stream.

positional arguments:
  message_name          Friendly name to remove/change this message later.

optional arguments:
  -h, --help            show this help message and exit
  -m MESSAGE, --message MESSAGE
                        Text to display on video stream scrolling msg
  -u URL, --url URL     URL of json RPC server to invoke commands on in form
                        IP:PORT.
  -f FONT, --font FONT  Pangocairo font family and size.
  -x XPOS, --xpos XPOS  Time in seconds to scroll text across screen.
  -y YPOS, --ypos YPOS  Vertical y pos of scrolling text in pixels.
  -d, --delete          Delte scrolling message via the provided message name.

```
An example of some Staic text can be seen below:
![Overlay Demo in VLC](https://raw.githubusercontent.com/on-three/VideoTextOverlay/e3a66d8a2a544106cd3198091f11d275a18979f8/img/Screenshot%20from%202014-04-28%2019:57:16.png)

## Text Markup
When overlaying text on a video stream, the text strings can be decorated with two kinds of markup:
* Pango Markup: To control text size, foreground and background color, font family, italic, bold etc
* strftime Markup: To insert current time information into the displayed string.

### strftime Markup
The current time can be inserted into any video stream overlay string by using pairs of curly braces enclosing normal strftime time format strings of the sort: `{{%Y-%m-%d %H:%M:%S}}`
For example, I want to display the current time in the upper right corner of the video stream, I could use the "static_text" python script (above) as follows:
```
~/code/video-text-relay $ ./python/static_msg "time_display" -m '{{%Y-%m-%d %H:%M:%S}}'  -s -b -x 475 -y 75
```
Note that the `-s` and `-b` switches above add a drop shadow and shaded underlay to make the text easier to read.

### Pango Markup
[Pango markup is described here](https://developer.gnome.org/pango/stable/PangoMarkupFormat.html). Markup of any sort can be injected into the displayed text. Markup is applied after the above time markup is applied.
For example, if I wanted to display the above upper right hand time display with extra large text and yellow, I could update the command as:
```
~/code/video-text-relay $ ./python/static_msg "one" -m '<span size="xx-large" foreground="yellow">{{%Y-%m-%d %H:%M:%S}}</span>'  -s -b -x 475 -y 75
```
If Pango markup parsing fails, the markup will be displayed on screen as plain text.

#Status and What Needs to Be Done:
I'm moving this forward, but still fairly primitive. Currently static and scrolling text displays are available, which can be added and removed via simple python JSON RPC clients.

I haven't yet put this into a debian or other package, so it must currently be built from source. I have yet to introduce a proper platform independent built script (like cmake) so the makefile will have to suffice.
As stated above, this is fairly primitive, but I believe I've confronted all major hurdles to showing text superimposed upon a relayed video stream.

The primary outstanding areas where work needs to be done are:
* ~~Currently these executables relay to local TCP clients (VLC via TCP connection provided by the GStreamer 'tcpsink' element). But a better model to stream to numerous remote clients may be needed. Perhaps 'tcpsink' need be replaced with the GStreamer shoutcast backend? Or their HTTP sink backend? This needs looking into.~~ As the application only outputs a TCP stream of the reencoded video, it currently needs to be further relayed via VLC (or some other applicaiton) to serve clients. To do this configure VLC to receive the video stream on 127.0.0.1:XXX and then provide an HTTP stream at whatever port you like. Multiple clients can then pick up the text overlaid stream.
* ~~I've only demonstrated a single RPC call that changes the text on the screen. I'd now have to (re) build code that allows the on-screen text to be manipulated in any number of ways: list boxes, time displays, text displays that use Pango Markup for text colors and weights.~~
* ~~The current code, though it works as a demo, is very poor quality. Needs a lot of work.~~
* ~~The JSON RPC mechanism cleanly separates relay servers from any possible client. But clients would have to be coded in some language to provide things like live IRC chats and other data.~~

