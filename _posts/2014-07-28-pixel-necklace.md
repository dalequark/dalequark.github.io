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

 My end goal in making this necklace was to have it alert me when my cellphone rang.  I didn't want to deal with convincing my phone to send out a bluetooth signal when I received a call, though, so I decided to have the LED light up when my phone emitted a high-frequency, ultrasonic pulse.  This was a pretty cheap solution, since it meant I only had to have a microphone connected to my microcontroller.  I also gave my pixel necklace a mode in which it lit up in response to distinct freqeuncy progressions (i.e. like those in Seven Nation Army).  

The code below, written for a Teensy 3.0 (which has a more powerful processer than Arduino's), computes the FFT of the input audio and lights up if the given sound transitions through notes in the desired way (i.e. in this case, plays Seven Nation Army).  There are more intelligent ways to do this than hard-code some upper and lower bounds for the tones, as you'll see below (todo: use Hidden Markov Models to train and recognize songs?). In any case, it works pretty well! Code to

    // The following code was taken from Adafruit and modified to display a rainbow pattern on neopixels in response to the distinctive Seven Nation Army riff. FFT implementation copyright 2013 [Tony DiCola (tony@tonydicola.com)] (http://learn.adafruit.com/fft-fun-with-fourier-transforms/).

    #define ARM_MATH_CM4
    #include <arm_math.h>
    #include <Adafruit_NeoPixel.h>


    ////////////////////////////////////////////////////////////////////////////////
    // CONIFIGURATION
    // These values can be changed to alter the behavior of the spectrum display.
    ////////////////////////////////////////////////////////////////////////////////

    int SAMPLE_RATE_HZ = 9000;             // Sample rate of the audio in hertz.


    // seven nation army high and low ranges
    const int TONE_HIGHS[]= {350, 350, 420, 355, 325, 285, 270, 350, 350, 420, 355};
    const int TONE_LOWS[] = {340, 340, 405, 345, 315, 275, 260, 340, 340, 405, 345};

    // threshold for turning on drum beats
    float BEAT_THRESHOLD_DB = 13.0;
    float LOWER_BEAT_FREQ = 10.0;
    float UPPER_BEAT_FREQ = 60.0;


    int TONE_ERROR_MARGIN_HZ = 10;         // Allowed fudge factor above and below the bounds for each tone input.
    int TONE_WINDOW_MS = 2000; //4000             // Maximum amount of milliseconds allowed to enter the full sequence.
    float TONE_THRESHOLD_DB = 10.0;        // Threshold (in decibels) each tone must be above other frequencies to count.
    const int FFT_SIZE = 256;              // Size of the FFT.  Realistically can only be at most 256
    // without running out of memory for buffers and other state.
    const int AUDIO_INPUT_PIN = 14;        // Input ADC pin for audio data.
    const int ANALOG_READ_RESOLUTION = 10; // Bits of resolution for the ADC.
    const int ANALOG_READ_AVERAGING = 16;  // Number of samples to average with each ADC reading.
    const int POWER_LED_PIN = 13;          // Output pin for power LED (pin 13 to use Teensy 3.0's onboard LED).
    const int NEO_PIXEL_PIN = 2;           // Output pin for neo pixels.
    const int NEO_PIXEL_COUNT = 1;         // Number of neo pixels.  You should be able to increase this without
    // any other changes to the program.
    const int MAX_CHARS = 65;              // Max size of the input command buffer


    ////////////////////////////////////////////////////////////////////////////////
    // INTERNAL STATE
    // These shouldn't be modified unless you know what you're doing.
    ////////////////////////////////////////////////////////////////////////////////

    IntervalTimer samplingTimer;
    float samples[FFT_SIZE*2];
    float magnitudes[FFT_SIZE];
    int sampleCounter = 0;
    Adafruit_NeoPixel pixels = Adafruit_NeoPixel(NEO_PIXEL_COUNT, NEO_PIXEL_PIN, NEO_GRB + NEO_KHZ800);
    char commandBuffer[MAX_CHARS];
    int tonePosition = 0;
    unsigned long toneStart = 0;


    ////////////////////////////////////////////////////////////////////////////////
    // MAIN SKETCH FUNCTIONS
    ////////////////////////////////////////////////////////////////////////////////

    void setup() {
      pinMode(10, OUTPUT);
      pinMode(8, OUTPUT);
      digitalWrite(8, LOW);
      digitalWrite(10, HIGH);
      // Set up serial port.
      Serial.begin(38400);

      // Set up ADC and audio input.
      pinMode(AUDIO_INPUT_PIN, INPUT);
      analogReadResolution(ANALOG_READ_RESOLUTION);
      analogReadAveraging(ANALOG_READ_AVERAGING);

      // Turn on the power indicator LED.
      pinMode(POWER_LED_PIN, OUTPUT);
      digitalWrite(POWER_LED_PIN, HIGH);

      // Initialize neo pixel library and turn off the LEDs
      pixels.begin();
      pixels.show();

      // Clear the input command buffer
      memset(commandBuffer, 0, sizeof(commandBuffer));

      // Begin sampling audio
      samplingBegin();
    }

    void loop() {
      // Calculate FFT if a full sample is available.
      if (samplingIsDone()) {
        // Run FFT on sample data.
        arm_cfft_radix4_instance_f32 fft_inst;
        arm_cfft_radix4_init_f32(&fft_inst, FFT_SIZE, 0, 1);
        arm_cfft_radix4_f32(&fft_inst, samples);
        // Calculate magnitude of complex numbers output by the FFT.
        arm_cmplx_mag_f32(samples, magnitudes, FFT_SIZE);

        // Detect tone sequence.
        //toneLoop();
        danceLoop(LOWER_BEAT_FREQ, UPPER_BEAT_FREQ);

        // Restart audio sampling.
        samplingBegin();
      }

      // Parse any pending commands.
      parserLoop();
    }


    ////////////////////////////////////////////////////////////////////////////////
    // UTILITY FUNCTIONS
    ////////////////////////////////////////////////////////////////////////////////

    // Compute the average magnitude of a target frequency window vs. all other frequencies.
    void windowMean(float* magnitudes, int lowBin, int highBin, float* windowMean, float* otherMean) {
      *windowMean = 0;
      *otherMean = 0;
      // Notice the first magnitude bin is skipped because it represents the
      // average power of the signal.
      for (int i = 1; i < FFT_SIZE/2; ++i) {
        if (i >= lowBin && i <= highBin) {
          *windowMean += magnitudes[i];
        }
        else {
          *otherMean += magnitudes[i];
        }
      }
      *windowMean /= (highBin - lowBin) + 1;
      *otherMean /= (FFT_SIZE / 2 - (highBin - lowBin));
    }

    // Convert a frequency to the appropriate FFT bin it will fall within.
    int frequencyToBin(float frequency) {
      float binFrequency = float(SAMPLE_RATE_HZ) / float(FFT_SIZE);
      return int(frequency / binFrequency);
    }

    // Convert intensity to decibels
    float intensityDb(float intensity) {
      return 20.0*log10(intensity);
    }

    // Compute the average magnitude of a target frequency window vs. all other frequencies.
    void windowMeanHighPass(float* magnitudes, int lowBin, int highBin, float* windowMean, float* otherMean, float upperBound) {
      *windowMean = 0;
      *otherMean = 0;
      int upperBoundBin;
      if(highBin <= upperBoundBin){
        upperBoundBin = frequencyToBin(upperBound);
      }
      else{
        upperBoundBin = FFT_SIZE/2;
      }
      // Notice the first magnitude bin is skipped because it represents the
      // average power of the signal.
      for (int i = 1; i < upperBoundBin; ++i) {
        if (i >= lowBin && i <= highBin) {
          *windowMean += magnitudes[i];
        }
        else {
          *otherMean += magnitudes[i];
        }
      }
      *windowMean /= (highBin - lowBin) + 1;
      *otherMean /= (upperBoundBin - (highBin - lowBin));
    }



    ////////////////////////////////////////////////////////////////////////////////
    // SPECTRUM DISPLAY FUNCTIONS
    ///////////////////////////////////////////////////////////////////////////////

    void toneLoop() {
      // Calculate the low and high frequency bins for the currently expected tone.
      int lowBin = frequencyToBin(TONE_LOWS[tonePosition] - TONE_ERROR_MARGIN_HZ);
      int highBin = frequencyToBin(TONE_HIGHS[tonePosition] + TONE_ERROR_MARGIN_HZ);
      // Get the average intensity of frequencies inside and outside the tone window.
      float window, other;
      windowMean(magnitudes, lowBin, highBin, &window, &other);
      window = intensityDb(window);
      other = intensityDb(other);
      // Check if tone intensity is above the threshold to detect a step in the sequence.
      if ((window - other) >= TONE_THRESHOLD_DB) {
        Serial.println("Value above threshold");
        // Start timing the window if this is the first in the sequence.
        unsigned long time = millis();
        if (tonePosition == 0) {
          toneStart = time;
        }
        // Increment key position if still within the window of key input time.
        if (toneStart + TONE_WINDOW_MS > time) {
          tonePosition += 1;
          Serial.println(tonePosition);
        }
        else {
          // Outside the window of key input time, reset back to the beginning key.
          tonePosition = 0;
        }
      }
      // Check if the entire sequence was passed through.
      if (tonePosition >= sizeof(TONE_LOWS)/sizeof(int)) {
        toneDetected();
        tonePosition = 0;
      }
    }

    void toneDetected() {
      // Flash the LEDs four times.
      int pause = 250;
      rainbow(0, 250);
      /*
      for (int i = 0; i < 4; ++i) {
        for (int j = 0; j < NEO_PIXEL_COUNT; ++j) {
          pixels.setPixelColor(j, pixels.Color(255, 0, 0));
        }
        pixels.show();
        delay(pause);
        for (int j = 0; j < NEO_PIXEL_COUNT; ++j) {
          pixels.setPixelColor(j, 0);
        }
        pixels.show();
        delay(pause);
      }
      */
    }

    // lowerFreq and upperFreq surround the "beat" of the music
    void danceLoop(float lowerFreq, float upperFreq){
      static int isOn = false;
      int pause = 250;
      // Calculate the low and high frequency bins for the currently expected tone.
      int lowBin = frequencyToBin(TONE_LOWS[tonePosition] - TONE_ERROR_MARGIN_HZ);
      int highBin = frequencyToBin(TONE_HIGHS[tonePosition] + TONE_ERROR_MARGIN_HZ);
      // Get the average intensity of frequencies inside and outside the tone window.
      float window, other;
      windowMean(magnitudes, lowBin, highBin, &window, &other);
      window = intensityDb(window);
      other = intensityDb(other);
      // Check if tone intensity is above the threshold to detect a step in the sequence.
      if ((window - other) >= BEAT_THRESHOLD_DB) {
        if(!isOn){
          pixels.setPixelColor(0, pixels.Color(0,200,200));
          isOn = true;
        }
        else{
          pixels.setPixelColor(0, pixels.Color(200,0,200));
          isOn = false;
        }  
        pixels.show();
        delay(pause);

      }
    }

    ////////////////////////////////////////////////////////////////////////////////
    // SAMPLING FUNCTIONS
    ////////////////////////////////////////////////////////////////////////////////

    void samplingCallback() {
      // Read from the ADC and store the sample data
      samples[sampleCounter] = (float32_t)analogRead(AUDIO_INPUT_PIN);
      // Complex FFT functions require a coefficient for the imaginary part of the input.
      // Since we only have real data, set this coefficient to zero.
      samples[sampleCounter+1] = 0.0;
      // Update sample buffer position and stop after the buffer is filled
      sampleCounter += 2;
      if (sampleCounter >= FFT_SIZE*2) {
        samplingTimer.end();
      }
    }

    void samplingBegin() {
      // Reset sample buffer position and start callback at necessary rate.
      sampleCounter = 0;
      samplingTimer.begin(samplingCallback, 1000000/SAMPLE_RATE_HZ);
    }

    boolean samplingIsDone() {
      return sampleCounter >= FFT_SIZE*2;
    }


    ////////////////////////////////////////////////////////////////////////////////
    // COMMAND PARSING FUNCTIONS
    // These functions allow parsing simple commands input on the serial port.
    // Commands allow reading and writing variables that control the device.
    //
    // All commands must end with a semicolon character.
    //
    // Example commands are:
    // GET SAMPLE_RATE_HZ;
    // - Get the sample rate of the device.
    // SET SAMPLE_RATE_HZ 400;
    // - Set the sample rate of the device to 400 hertz.
    //
    ////////////////////////////////////////////////////////////////////////////////
    void rainbow(int thisPixel, int timeToRainbow){
      for(int i=0; i < timeToRainbow; i++){
        pixels.setPixelColor(thisPixel, Wheel(i) % 255);
      }
      pixels.show();
      pixels.wait(10)
    }

    // Input a value 0 to 255 to get a color value.
    // The colours are a transition r - g - b - back to r.
    uint32_t Wheel(byte WheelPos) {
      if(WheelPos < 85) {
        return pixels.Color(WheelPos * 3, 255 - WheelPos * 3, 0);
        } else if(WheelPos < 170) {
          WheelPos -= 85;
          return pixels.Color(255 - WheelPos * 3, 0, WheelPos * 3);
          } else {
            WheelPos -= 170;
            return pixels.Color(0, WheelPos * 3, 255 - WheelPos * 3);
          }
        }


        void parserLoop() {
          // Process any incoming characters from the serial port
          while (Serial.available() > 0) {
            char c = Serial.read();
            // Add any characters that aren't the end of a command (semicolon) to the input buffer.
            if (c != ';') {
              c = toupper(c);
              strncat(commandBuffer, &c, 1);
            }
            else
            {
              // Parse the command because an end of command token was encountered.
              parseCommand(commandBuffer);
              // Clear the input buffer
              memset(commandBuffer, 0, sizeof(commandBuffer));
            }
          }
        }

        // Macro used in parseCommand function to simplify parsing get and set commands for a variable
        #define GET_AND_SET(variableName) \
        else if (strcmp(command, "GET " #variableName) == 0) { \
          Serial.println(variableName); \
          } \
          else if (strstr(command, "SET " #variableName " ") != NULL) { \
            variableName = (typeof(variableName)) atof(command+(sizeof("SET " #variableName " ")-1)); \
          }

          void parseCommand(char* command) {
            if (strcmp(command, "GET MAGNITUDES") == 0) {
              for (int i = 0; i < FFT_SIZE; ++i) {
                Serial.println(magnitudes[i]);
              }
            }
            else if (strcmp(command, "GET SAMPLES") == 0) {
              for (int i = 0; i < FFT_SIZE*2; i+=2) {
                Serial.println(samples[i]);
              }
            }
            else if (strcmp(command, "GET FFT_SIZE") == 0) {
              Serial.println(FFT_SIZE);
            }
            GET_AND_SET(SAMPLE_RATE_HZ)
            GET_AND_SET(TONE_ERROR_MARGIN_HZ)
            GET_AND_SET(TONE_WINDOW_MS)
            GET_AND_SET(TONE_THRESHOLD_DB)
          }


 The following code lights up an LED when a tone of 19.23khz (just above the audible range for most people). In the
last example, I used a FFT to identify peak frequencies in audio input.
Unfortunately, I found the Arduino Uno wasn't beefy enough to run an FFT fast enough to identify
a frequency as high as 20khz.  Instead, I found a great  [post by Rob B](http://blog.theultimatelabs.com/2013/05/wirelessly-communicating-with-arduino.html) explaining how to use
the [Goertzel Algorithm](http://en.wikipedia.org/wiki/Goertzel_algorithm) for detecting a single frequency.
Rob also discovered a neat trick about the Goertzel algorithm, which is that if one chooses
the coefficients of the alogrithm to be 0 (making the Goertzel transform much faster,
since we need not compute multiplication by 0), the frequency it detects is 19.23khz, just above
the audible range. Check out the implementation below:


     // Ultrasonic Communication Sketch

     #include <avr/pgmspace.h>
     #include <Adafruit_NeoPixel.h>

     boolean ready = 0;

     // Pin for a neopixel or a neopixel strip
     #define PIN 6
     int BUILTINPIN = 13;
     #define ADC_CENTER 127
     #define WINDOW_SIZE 128
     #define BIT_PERIOD 8
     #define BIT_PERIOD_2 16
     #define BIT_PERIOD_5_2 20
     #define BIT_PERIOD_3_2 12
     #define BIT_PERIOD_1_2 4
     #define THRESHOLD 25000

     unsigned long mag;
     int counter = 0;
     const unsigned int ON_THRESHOLD = 500;
     const unsigned int OFF_THRESHOLD = ON_THRESHOLD/2;


     const float SAMPLE_RATE = 76923.0769;
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
     Adafruit_NeoPixel strip = Adafruit_NeoPixel(1, PIN, NEO_GRB + NEO_KHZ800);

     void setup() {

        strip.begin();
        strip.show(); // Initialize all pixels to 'off'

       Serial.begin(9600); // Serial for debugging
       initADC();



     }

     int tcount = 0;
     void loop()
     {
       // Every time a measurement is taken above the given threshold,
       // counter is incremented. If counter is above the threshold for
       // 10 consecutive seconds, show some colors.
       if(counter > 10)
       {
         rainbowCycle(5);
         // After, turn of neopixel
         strip.setPixelColor(0,0,0,0);
         strip.show();
       }

     }

      // Enable interrupt-driven measurements to make things faster
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

       // Interrupt service routine is called each time a measurement is taken
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
