# ArduinoOSC
This code is designed to connect an Arduino to the ETC EOS control system.  Connection is via USB and messages are sent
using Open Show Control (OSC) protocol.

Testing was done with a temperature probe and a sound sensor trigger, and was used to control Hue, Saturation and Intensity
of a single channel on the lighting console.

I've found that the format for the OSC message being sent is VERY particular and is the most frustrating part of the project.

Credit must be given to the ETCLabs and ETC Lighthack repositories on Github that were instrumental in getting me the building
blocks to get this code up and running.

Scale would be the next step-  multiple sensors controlling multiple lights.

Code given below

*******************************************************************************
 * Includes
 ******************************************************************************/
#include <OSCBoards.h>
#include <OSCBundle.h>
#include <OSCData.h>
#include <OSCMatch.h>
#include <OSCMessage.h>
#include <OSCTiming.h>

#include <OneWire.h>
#include <DallasTemperature.h>

#ifdef BOARD_HAS_USB_SERIAL
#include <SLIPEncodedUSBSerial.h>
SLIPEncodedUSBSerial SLIPSerial(thisBoardsSerialUSB);
#else
#include <SLIPEncodedSerial.h>
SLIPEncodedSerial SLIPSerial(Serial);
#endif
#include <string.h>

 /*******************************************************************************
  * Macros and Constants
  ******************************************************************************/

#define OSC_BUF_MAX_SIZE    512

const String HANDSHAKE_QUERY = "ETCOSC?";
const String HANDSHAKE_REPLY = "OK";

//See displayScreen() below - limited to 10 chars (after 6 prefix chars)
const String VERSION_STRING = "1.0.0.5";



#define ONE_WIRE_BUS 2  //Data wire is plugged into pin 2 on Arduino

OneWire oneWire(ONE_WIRE_BUS); 
  //Setup a oneWire instance to communicate with any OneWire device

DallasTemperature sensors(&oneWire);
/*******************************************************************************
 * Local Types
 ******************************************************************************/


/*******************************************************************************
 * Global Variables
 ******************************************************************************/
int CurrentTemp = 0;     //variable for temp
int SatValue = 0;       //variable for Saturation
int Hue = 0;           //variable for Hue
int Brightness = 30;  //starting brightness value
int moreFuego = 10;  //amount intensity up change
int Dimness = 2;  //amount auto down intensity change
int Chan = 11;  //channel affected.  Effected?


const int MIC = 3;  //input from button/mic sensor

unsigned long elapseLimit = 3000;  //amount of time before dimness
unsigned long PrevTime = 0;  //restart point for counter


bool MicState = false;
bool ledState = false;  //state variable for the LED

bool updateDisplay = false;
bool connectedToEos = false;
unsigned long lastMessageRxTime = 0;
bool timeoutPingSent = false;

/*******************************************************************************
 * Local Functions
 ******************************************************************************/

/*******************************************************************************
 * Issues all our subscribes to Eos. When subscribed, Eos will keep us updated
 * with the latest values for a given parameter.
 * Kept here for reference, not necessary if all information is designed to travel
 * in only one direction.
 *
 * Parameters:  none
 *
 * Return Value: void
 *
 ******************************************************************************/
void issueSubscribes()
{
    // Add a filter so we don't get spammed with unwanted OSC messages from Eos
    OSCMessage filter("/eos/filter/add");
    filter.add("/eos/out/param/*");
    filter.add("/eos/out/ping");
    SLIPSerial.beginPacket();
    filter.send(SLIPSerial);
    SLIPSerial.endPacket();

 
}



/*******************************************************************************
 * Given an unknown OSC message we check to see if it's a handshake message.
 * If it's a handshake we issue a subscribe, otherwise we begin route the OSC
 * message to the appropriate function.
 *
 * Parameters:
 *  msg - The OSC message of unknown importance
 *
 * Return Value: void
 *
 ******************************************************************************/
void parseOSCMessage(String& msg)
{
    // check to see if this is the handshake string
    if (msg.indexOf(HANDSHAKE_QUERY) != -1)
    {
        // handshake string found!
        SLIPSerial.beginPacket();
        SLIPSerial.write((const uint8_t*)HANDSHAKE_REPLY.c_str(), (size_t)HANDSHAKE_REPLY.length());
        SLIPSerial.endPacket();

        // Let Eos know we want updates on some things
        issueSubscribes();

        // Make our splash screen go away
        connectedToEos = true;
        updateDisplay = true;
    }
    else
    {
        // prepare the message for routing by filling an OSCMessage object with our message string
        OSCMessage oscmsg;
        oscmsg.fill((uint8_t*)msg.c_str(), (int)msg.length());
        
    }
}


/*******************************************************************************
 * Here we setup our encoder, lcd, and various input devices. We also prepare
 * to communicate OSC with Eos by setting up SLIPSerial. Once we are done with
 * setup() we pass control over to loop() and never call setup() again.
 *
 * NOTE: This function is the entry function. This is where control over the
 * Arduino is passed to us (the end user).
 *
 * Parameters: none
 *
 * Return Value: void
 *
 ******************************************************************************/
void setup()
{
    SLIPSerial.begin(115200);
    // This is a hack around an Arduino bug. It was taken from the OSC library
    //examples
#ifdef BOARD_HAS_USB_SERIAL
    while (!SerialUSB);
#else
    while (!Serial);
#endif
pinMode(MIC, INPUT);  //input from sound sensor to pin 3

//pinMode(13, OUTPUT);  //flash LED if sound triggers for debug
//digitalWrite(13, ledState);  //sound sensor has onboard LED, so not really necessary

sensors.begin();


    // This is necessary for reconnecting a device because it needs some time
    // for the serial port to open, but meanwhile the handshake message was
    // sent from Eos
    SLIPSerial.beginPacket();
    SLIPSerial.write((const uint8_t*)HANDSHAKE_REPLY.c_str(), (size_t)HANDSHAKE_REPLY.length());
    SLIPSerial.endPacket();
    // Let Eos know we want updates on some things
    issueSubscribes();

delay(1000);

 OSCMessage on("/eos/chan/11/at/");  //set initial starting intensity on channel 11
              on.add(Brightness);
              
              SLIPSerial.beginPacket(); //send the thing
              on.send(SLIPSerial);
              SLIPSerial.endPacket();
              
              delay(1000); 
              
}

/*******************************************************************************
 * Here we service, monitor, and otherwise control all our peripheral devices.
 * First, we retrieve the status of our encoders and buttons and update Eos.
 * Then we apply any transformation functions and output the data to Eos.

 *
 * NOTE: This function is our main loop and thus this function will be called
 * repeatedly forever
 *
 * Parameters: none
 *
 * Return Value: void
 *
 ******************************************************************************/
void loop()
{
 unsigned long currentTime = millis();  //checks current time
 boolean timeOut = false;
    
    static String curMsg;
    int size;
    // get the updated state of each encoder

MicState = digitalRead(MIC);  //check for sensor triggered
  if (MicState == HIGH) {
    Brightness = Brightness+moreFuego;  //change intensity output by current brightness + step
    Brightness = constrain(Brightness, 0, 100);  //keeps numbers in bounds
  }

sensors.requestTemperatures(); //queries temperature probe
CurrentTemp = sensors.getTempCByIndex(0);  //converts to degrees Celcius.
//Change C to F for the other system.
CurrentTemp = constrain(CurrentTemp, 0, 40);  
//constrains to between 0 and 40, ensures we stay within bounds and don't
//exceed workable values when converting to Hue and Saturation
  
  if (CurrentTemp >= 24)  //we are in the hot zone
    {
      Hue = 360;
      SatValue = (CurrentTemp-24)*5;  //scale our temp into Saturation
    }
   else  //we are in the cold zone
    {
      Hue = 240; 
      SatValue = (24-CurrentTemp)*4;  //scale our temp into Saturation
     }
      

      
      OSCMessage stuff("/eos/cs/color/hs/");  //prep EOS to receive colour
              stuff.add(Hue);  //add Hue variable
              stuff.add(SatValue);  //add Saturation variable
              
              SLIPSerial.beginPacket();  //send the thing
              stuff.send(SLIPSerial);
              SLIPSerial.endPacket();

           delay(10);

         if(MicState == HIGH) {  //if intensity trigger was triggered
          OSCMessage intensity("/eos/at/");  //prep EOS for intensity value
          intensity.add(Brightness);  //add brightness variable to message
          
          SLIPSerial.beginPacket();  //send the thing
          intensity.send(SLIPSerial);
          SLIPSerial.endPacket();

          PrevTime = currentTime;  //a message was sent, so reset our timer
         } else {
          delay(1);
         }
        
//check to see if long enough to lower intensity
    
if(currentTime-PrevTime>elapseLimit){  //compares time values, if greater than limit, run intensity down
  Brightness = Brightness-Dimness;  //subtracts our down step from current intensity level
  if (Brightness <= 30) {  //sets our minimum intensity level, also keeps number from going negative.
    Brightness = 30;
  }
  
  OSCMessage downboy("/eos/at");  //our intensity down section
  downboy.add(Brightness);  //adds intensity value to message
  SLIPSerial.beginPacket();  //send the thing
  downboy.send(SLIPSerial);
  SLIPSerial.endPacket();

  PrevTime = currentTime;  //a message was sent, so we want to reset our timer
}  else {
delay(1);
}     

   
    
}
