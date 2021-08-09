# task-4-

تقوم المهمة الرابعة على تحويل الصوت ( الكلام ) الى نص ( text ) وقمت بالخطوات التالية : 
بداية قمت بانشاء حساب على cloud.ibm.com  لاستخدام Watson service  ومن ثم بعد ذلك قمت باختيار catalog من الموقع واختيار AI / machine learning   وبعد ذلك قمت باختيار خدمة التحويل speech to text  ومن ثم قمت باختيار region وهو Sydney(au-syd) . وبعد ذلك قمت بانشاءه بالضغط على create  ثم service credentials  وبعد الضغط على auto-generated  سيظهر لنا الكود الذي نحتاجه وهو كالتالي : 
{
  "apikey": "h9fZAuWkManDDLieUSeoq2__GqDdnt87pe_OMyZWdKRr",
  "iam_apikey_description": "Auto-generated for key 8c091806-30bc-4d1a-a7c9-c43c1383a8da",
  "iam_apikey_name": "Auto-generated service credentials",
  "iam_role_crn": "crn:v1:bluemix:public:iam::::serviceRole:Manager",
  "iam_serviceid_crn": "crn:v1:bluemix:public:iam-identity::a/1127bb7a842a497c9e53e43f4caa3c7a::serviceid:ServiceId-1c4c561b-24fe-43a0-84ba-fefa9e97d4b6",
  "url": "https://api.au-syd.speech-to-text.watson.cloud.ibm.com/instances/94d9d825-8357-4969-9ba2-da119d919ce2"
}

وقمت بتحميل النسخة الثالثة من python  لعمل الكود عليها .  
ثم قمت بادخال الكود التالي على برنامج visual studio code  : 
[metadata]
	name = watson_sstt
	summary = Watson Streaming Speech to Text
	description-file =
	    README.rst
	author = Sean Dague
	author-email = sean.dague@ibm.com
	classifier =
	    Intended Audience :: Information Technology
	    License :: OSI Approved :: Apache Software License
	    Operating System :: POSIX :: Linux
	    Programming Language :: Python
	    Programming Language :: Python :: 3
	    Programming Language :: Python :: 3.5
	

	[files]
	packages =
	    watson_sstt
	

	[build_sphinx]
	source-dir = doc/source
	build-dir = doc/build
	all_files = 1
	

	[upload_sphinx]
	upload-dir = doc/build/html
وايضا الكود التالي في ملف اخر : 
import multiprocessing  # noqa
	except ImportError:
	    pass
	

	setuptools.setup(
	    setup_requires=['pbr>=1.8'],
	    pbr=True)

ومن ثم قمت بوضع المعلومات الخاصة بي وهي : 
"apikey": "h9fZAuWkManDDLieUSeoq2__GqDdnt87pe_OMyZWdKRr"
Region : au-syd


وبعد ذلك استخدمت الكود : 
def read_audio(ws, timeout):
	    """Read audio and sent it to the websocket port.
	
	    This uses pyaudio to read from a device in chunks and send these
	    over the websocket wire.
	
	    """
	    global RATE
	    p = pyaudio.PyAudio()
	    # NOTE(sdague): if you don't seem to be getting anything off of
	    # this you might need to specify:
	    #
	    #    input_device_index=N,
	    #
	    # Where N is an int. You'll need to do a dump of your input
	    # devices to figure out which one you want.
	    RATE = int(p.get_default_input_device_info()['defaultSampleRate'])
	    stream = p.open(format=FORMAT,
	                    channels=CHANNELS,
	                    rate=RATE,
	                    input=True,
	                    frames_per_buffer=CHUNK)
	

	    print("* recording")
	    rec = timeout or RECORD_SECONDS
	

	    for i in range(0, int(RATE / CHUNK * rec)):
	        data = stream.read(CHUNK)
	        # print("Sending packet... %d" % i)
	        # NOTE(sdague): we're sending raw binary in the stream, we
	        # need to indicate that otherwise the stream service
	        # interprets this as text control messages.
	        ws.send(data, ABNF.OPCODE_BINARY)
	

	    # Disconnect the audio stream
	    stream.stop_stream()
	    stream.close()
	    print("* done recording")
	

	    # In order to get a final response from STT we send a stop, this
	    # will force a final=True return message.
	    data = {"action": "stop"}
	    ws.send(json.dumps(data).encode('utf8'))
	    # ... which we need to wait for before we shutdown the websocket
	    time.sleep(1)
	    ws.close()
	

	    # ... and kill the audio device
	    p.terminate()
	

	

	def on_message(self, msg):
	    """Print whatever messages come in.
	
	    While we are processing any non trivial stream of speech Watson
	    will start chunking results into bits of transcripts that it
	    considers "final", and start on a new stretch. It's not always
	    clear why it does this. However, it means that as we are
	    processing text, any time we see a final chunk, we need to save it
	    off for later.
	    """
	    global LAST
	    data = json.loads(msg)
	    if "results" in data:
	        if data["results"][0]["final"]:
	            FINALS.append(data)
	            LAST = None
	        else:
	            LAST = data
	        # This prints out the current fragment that we are working on
	        print(data['results'][0]['alternatives'][0]['transcript'])
	

	

	def on_error(self, error):
	    """Print any errors."""
	    print(error)
	

	

	def on_close(ws):
	    """Upon close, print the complete and final transcript."""
	    global LAST
	    if LAST:
	        FINALS.append(LAST)
	    transcript = "".join([x['results'][0]['alternatives'][0]['transcript']
	                          for x in FINALS])
	    print(transcript)
	

	

	def on_open(ws):
	    """Triggered as soon a we have an active connection."""
	    args = ws.args
	    data = {
	        "action": "start",
	        # this means we get to send it straight raw sampling
	        "content-type": "audio/l16;rate=%d" % RATE,
	        "continuous": True,
	        "interim_results": True,
	        # "inactivity_timeout": 5, # in order to use this effectively
	        # you need other tests to handle what happens if the socket is
	        # closed by the server.
	        "word_confidence": True,
	        "timestamps": True,
	        "max_alternatives": 3
	    }
	

	    # Send the initial control message which sets expectations for the
	    # binary stream that follows:
	    ws.send(json.dumps(data).encode('utf8'))
	    # Spin off a dedicated thread where we are going to read and
	    # stream out audio.
	    threading.Thread(target=read_audio,
	                     args=(ws, args.timeout)).start()
	

	def get_url():
	    config = configparser.RawConfigParser()
	    config.read('speech.cfg')
	    # See
	    # https://console.bluemix.net/docs/services/speech-to-text/websockets.html#websockets
	    # for details on which endpoints are for each region.
	    region = config.get('auth', 'region')
	    host = REGION_MAP[region]
	    return ("wss://{}/speech-to-text/api/v1/recognize"
	           "?model=en-AU_BroadbandModel").format(host)
	

	def get_auth():
	    config = configparser.RawConfigParser()
	    config.read('speech.cfg')
	    apikey = config.get('auth', 'apikey')
	    return ("apikey", apikey)
	

	

	def parse_args():
	    parser = argparse.ArgumentParser(
	        description='Transcribe Watson text in real time')
	    parser.add_argument('-t', '--timeout', type=int, default=5)
	    # parser.add_argument('-d', '--device')
	    # parser.add_argument('-v', '--verbose', action='store_true')
	    args = parser.parse_args()
	    return args
	

	

	def main():
	    # Connect to websocket interfaces
	    headers = {}
	    userpass = ":".join(get_auth())
	    headers["Authorization"] = "Basic " + base64.b64encode(
	        userpass.encode()).decode()
	    url = get_url()
	

	    # If you really want to see everything going across the wire,
	    # uncomment this. However realize the trace is going to also do
	    # things like dump the binary sound packets in text in the
	    # console.
	    #
	    # websocket.enableTrace(True)
	    ws = websocket.WebSocketApp(url,
	                                header=headers,
	                                on_message=on_message,
	                                on_error=on_error,
	                                on_close=on_close)
	    ws.on_open = on_open
	    ws.args = parse_args()
	    # This gives control over the WebSocketApp. This is a blocking
	    # call, so it won't return until the ws.close() gets called (after
	    # 6 seconds in the dedicated thread).
	    ws.run_forever()
	

	

	if __name__ == "__main__":
	    main()

ومن ثم بعد ذلك قمت باستخدام jupyter لتحويل الصوت الى نص وقمت باستخدام الكود التالي : 
In [1]:
!pip install ibm_watson
Requirement already satisfied: ibm_watson in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (4.5.0)
Requirement already satisfied: websocket-client==0.48.0 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from ibm_watson) (0.48.0)
Requirement already satisfied: requests<3.0,>=2.0 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from ibm_watson) (2.22.0)
Requirement already satisfied: ibm-cloud-sdk-core==1.5.1 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from ibm_watson) (1.5.1)
Requirement already satisfied: python-dateutil>=2.5.3 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from ibm_watson) (2.8.0)
Requirement already satisfied: six in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from websocket-client==0.48.0->ibm_watson) (1.12.0)
Requirement already satisfied: certifi>=2017.4.17 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from requests<3.0,>=2.0->ibm_watson) (2019.9.11)
Requirement already satisfied: idna<2.9,>=2.5 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from requests<3.0,>=2.0->ibm_watson) (2.8)
Requirement already satisfied: urllib3!=1.25.0,!=1.25.1,<1.26,>=1.21.1 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from requests<3.0,>=2.0->ibm_watson) (1.24.2)
Requirement already satisfied: chardet<3.1.0,>=3.0.2 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from requests<3.0,>=2.0->ibm_watson) (3.0.4)
Requirement already satisfied: PyJWT>=1.7.1 in /Users/nicholasrenotte/opt/anaconda3/lib/python3.7/site-packages (from ibm-cloud-sdk-core==1.5.1->ibm_watson) (1.7.1)
In [10]:
from ibm_watson import SpeechToTextV1
from ibm_watson.websocket import RecognizeCallback, AudioSource 
from ibm_cloud_sdk_core.authenticators import IAMAuthenticator
In [11]:
apikey = ' h9fZAuWkManDDLieUSeoq2__GqDdnt87pe_OMyZWdKRr
'
url = ' https://api.au-syd.speech-to-text.watson.cloud.ibm.com/instances/94d9d825-8357-4969-9ba2-da119d919ce2
'
with open('Untitled.mp3', 'rb') as f:
    res = stt.recognize(audio=f, content_type='audio/mp3', model='en-US_NarrowbandModel', continuous=True).get_result()
{'result_index': 0,
 'results': [{'final': True,
   'alternatives': [{'transcript': 'hello well ', 'confidence': 0.8}]}]}
text = res['results'][0]['alternatives'][0]['transcript']
text
confidence = res['results'][0]['alternatives'][0]['confidence']
confidence
with open('output.txt', 'w') as out:
    out.writelines(text)
with open('Untitled.mp3', 'rb') as f:
    res = stt.recognize(audio=f, content_type='audio/mp3', model='en-AU_NarrowbandModel', continuous=True).get_result()
{'result_index': 0,
 'results': [{'final': True,
   'alternatives': [{'transcript': 'hello world ', 'confidence': 0.99}]}]}
وقمت بالتعديل عليها وفقا لمعلوماتي . 


