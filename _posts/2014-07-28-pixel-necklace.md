---
layout: article
title: Infrasonic Pixel Necklace
tags: [Electronics]
excerpt: "A necklace that glows up in response to infrasonic signals or music,
made from a Neopixel, microcontroller, and headphone cables. Featured on Adafruit!"
image:
  feature:
  teaser: glowynecklace.gif
---

This project was featured on the [Adafruit Wearables Blog](http://www.adafruit.com/blog/2014/11/19/glowy-necklace-made-from-headphone-cable-wearablewednesday/), and was a Featured Article on [Instructables](ttp://www.instructables.com/id/Electronics-Necklace-from-Headphone-Cable/?embed=flash).

This pixel necklace was definitely my favorite project, mostly because it has quite a few design tricks up its sleeve.

 First, as far as connected necklaces go, it has a pretty small pendant with a powerful processor.  The trick is that the microcontroller controlling the tri-color LED pendant is actually hid behind my neck, and controls the LED via the super-thin wires in a headphone cable.  The clasp for the necklace is a headphone cable jack; I close the necklace by snapping the headphone cable jack into its socket, which also serves as a switch for the necklace.  You can read about how I worked with headphone cables in the [Instructable](ttp://www.instructables.com/id/Electronics-Necklace-from-Headphone-Cable/?embed=flash).  

 My end goal in making this necklace was to have it alert me when my cellphone rang.  I didn't want to deal with convincing my phone to send out a bluetooth signal when I received a call, though, so I decided to have the LED light up when my phone emitted a high-frequency, ultrasonic pulse.  This was a pretty cheap solution, since it meant I only had to have a microphone connected to my microcontroller.  I also gave my pixel necklace a mode in which it lit up in response to distinct freqeuncy progressions (i.e. like those in Seven Nation Army).  Below is some code to do this that I haven't looked at for months, that was informed by [this code](http://blog.theultimatelabs.com/2013/05/wirelessly-communicating-with-arduino.html), so I'm sorry if it's nonsensical.  
 //TODO: Make sense of this code



     #include <avr/pgmspace.h>
     #include <Adafruit_NeoPixel.h>

     boolean ready = 0;
     #define PIN 6
     int BUILTINPIN = 13;
     #define ADC_CENTER 127
     #define WINDOW_SIZE 128
     #define BIT_PERIOD 8
     #define BIT_PERIOD_2 16
     #define BIT_PERIOD_5_2 20
     #define BIT_PERIOD_3_2 12
     #define BIT_PERIOD_1_2 4
     #define MSG_LENGTH 3
     #define NUM_BYTES 5
     #define NUM_MSGS 5
     #define THRESHOLD 25000

     unsigned long mag;
     int counter = 0;
     const unsigned int ON_THRESHOLD = 500;
     const unsigned int OFF_THRESHOLD = ON_THRESHOLD/2;


     const float SAMPLE_RATE = 76923.0769;//38461.5385 //8928.57143
     const float STAMP_MS = (float)WINDOW_SIZE/(SAMPLE_RATE/1000);

     byte bytei = 0;
     byte biti = 0;
     byte lastSmpl = 0;
     byte bankIn = 0;
     byte bankProc = 0;

     unsigned short samplei=0;
     unsigned long stamp=0;
     unsigned long lastStamp=0;


     int Q1,Q2;
     //Adafruit_NeoPixel strip = Adafruit_NeoPixel(1, PIN, NEO_GRB + NEO_KHZ800);

     void setup() {

       // strip.begin();
       //strip.show(); // Initialize all pixels to 'off'

       Serial.begin(9600);
       initADC();



     }

     int tcount = 0;
     void loop()
     {
       tcount++;
       Serial.print("Count is ");
       Serial.println(counter);
       /*
       if(counter > 10)
       {
         rainbowCycle(5);
         // After, turn of neopixel
         strip.setPixelColor(0,0,0,0);
         strip.show();
       }
       */

     }

     // Slightly different, this makes the rainbow equally distributed throughout
     /*
     void rainbowCycle(uint8_t wait) {
       uint16_t i, j;

       for(j=0; j<256*5; j++) { // 5 cycles of all colors on wheel
         for(i=0; i< strip.numPixels(); i++) {
           strip.setPixelColor(i, Wheel(((i * 256 / strip.numPixels()) + j) & 255));
         }
         strip.show();
         delay(wait);
       }
     }
     */
     // Input a value 0 to 255 to get a color value.
     // The colours are a transition r - g - b - back to r.
     /*
     uint32_t Wheel(byte WheelPos) {
       if(WheelPos < 85) {
         return strip.Color(WheelPos * 3, 255 - WheelPos * 3, 0);
         } else if(WheelPos < 170) {
           WheelPos -= 85;
           return strip.Color(255 - WheelPos * 3, 0, WheelPos * 3);
           } else {
             WheelPos -= 170;
             return strip.Color(0, WheelPos * 3, 255 - WheelPos * 3);
           }
         }

         */
         void initADC() {
           cli();//diable interrupts

           //set up continuous sampling of analog pin 0

           //clear ADCSRA and ADCSRB registers
           ADCSRA = 0;
           ADCSRB = 0;

           ADMUX |= (1 << REFS0); //set reference voltage
           ADMUX |= (1 << ADLAR); //left align the ADC value- so we can read highest 8 bits from ADCH register only

           ADCSRA |= (1 << ADPS2) ;//| (1 << ADPS0); //set ADC clock with 32 prescaler- 16mHz/32=500kHz
           ADCSRA |= (1 << ADATE); //enabble auto trigger
           enableADCInterrupt();//ADCSRA |= (1 << ADIE); //enable interrupts when measurement complete
           ADCSRA |= (1 << ADEN); //enable ADC
           ADCSRA |= (1 << ADSC); //start ADC measurements

           sei();//enable interrupts
         }

         void enableADCInterrupt() {
           ADCSRA |= (1 << ADIE); //enable interrupts when measurement complete
         }
         void disableADCInterrupt() {
           ADCSRA &= ~(1 << ADIE); //enable interrupts when measurement complete
         }


         ISR(ADC_vect) {
           int Q0 = -Q2 + ((int)ADCH-ADC_CENTER);
           Q2 = Q1;
           Q1 = Q0;

           samplei++;
           if(samplei == WINDOW_SIZE) {
             samplei = 0;
             unsigned long mag = (unsigned long)((long)Q1)*Q1+(unsigned long)((long)Q2)*Q2;

             Q1=Q2=0;

             if(mag > ON_THRESHOLD) {
               counter = 1;
             }
             else if (mag < OFF_THRESHOLD) {
               counter = 0;
             }
             else
             {
               counter = 3;
             }
           }
         }
