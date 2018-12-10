# ArduinoOSC
This code is designed to connect an Arduino to the ETC EOS control system.  Connection is via USB and messages are sent
using Open Show Control (OSC) protocol.

Testing was done with a temperature probe and a sound sensor trigger, and was used to control Hue, Saturation and Intensity
of a single channel on the lighting console.

I've found that the format for the OSC message being sent is VERY particular and is the most frustrating part of the project.

Credit must be given to the ETCLabs and ETC Lighthack repositories on Github that were instrumental in getting me the building
blocks to get this code up and running.

Scale would be the next step-  multiple sensors controlling multiple lights.
