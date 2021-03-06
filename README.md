# Arduino UNO R3-based LC-meter

 Author: Alexander Popov, 2018.
   
 alexander.popov9@mail.ru
  
  
    
 ### Schematics 
 
 Licensed under Creative Commons Attribution Share-Alike license.
  
 https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/LC_meter.pdf
  
  ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/Screenshot%20from%202019-01-05%2003-31-01.png "Schematics")
 
  
  ### Sketch 
  
  Licensed under GPL V3.
  
  https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/LC_meter.ino
  
  ```
  /* LC-meter powered by Arduino.
   Author Alexander Popov, 2018 alexander.popov9@mail.ru
   Licensed under GPL V3.

   This sketch is a compilation from various free sources one can find on the Internet and the original code.
   Thanks to all original authors and  contributors.
   
   This sketch was tested with Arduino UNO R3.

   Functionality:

   1. IR Remote Control Tester.
   
   Based on:
   https://www.instructables.com/id/Analyze-Any-IR-Protocol-With-Just-You-Arduino-Boar/

   2. Frequency measurement.

   Based on:
   http://interface.khm.de/index.php/lab/interfaces-advanced/arduino-frequency-counter-library/

   3.  Inductance meter.

   Based on:
   https://0jihad0.livejournal.com/2081.html
   https://www.radiolocman.com/shem/schematics.html?di=162628

   4. Capacitance meter.

   Based on:
   https://www.arduino.cc/en/Tutorial/CapacitanceMeter

   5. Mode selection is an original code.

*/ 

#include <Wire.h>
#include <LiquidCrystal_I2C.h>      // https://github.com/johnrickman/LiquidCrystal_I2C
#include <FreqCounter.h>            // http://interface.khm.de/wp-content/uploads/2009/01/FreqCounter_1_12.zip

// PIN ASSIGNMENTS

// MODE SELECTOR:

#define pin_mode_LM  2            // digital pin 2,  HIGH, frequency measurement 
#define pin_mode_LS  4            // digital pin 4,  HIGH, inductance measurement
#define pin_mode_CL  9            // digital pin 9,  HIGH, capacitance measurement, large mode 
#define pin_mode_CS 10            // digital pin 10, HIGH, capacitance measurement, small mode

// IR Remote Control Tester

#define IR_PIN 3 

// Frequency measurement & Inductance meter:
// digital pin 5

// Capacitance meter:

#define analogPin      0          // analog pin for measuring capacitor voltage
#define chargePin      13         // digital pin to charge the capacitor - connected to one end of the charging resistor
#define dischargePin   11         // digital pin to discharge the capacitor

// Variables for MODE Selection:

int mode =  0;   // measuring mode: 0 - IR remote control tester
                 //                 1 - frequency measurement
                 //                 2 - inductance measurement
                 //                 4 - capacitance measurement, large mode
                 //                 8 - capacitance measurement, small mode

// Variables used in IR remote control tester:

volatile unsigned int  now, start, capture[3], i, add = 0;
volatile boolean flag_complete;

// Variables used in frequency&inductance meter:
               
double pi = 3.14159;
double frq = 0;
double frequency;                      // measure frequency using the pulse as in usec use 1.E6
double inductance;         
double cap = 1475e-12;                 // ~1500 pF, for measuring  inductance           
float correction_inductance = 0.67;    // correction inductance, uH
char *inductance_units = "uH";
double prescaler = 16;

// Variables used in capacitance meter:

int new_mode = 0;                                                                  
double resistorValue1 = 9620.0F;      // ~10k,  for measuring large capacitance
double resistorValue2 = 87940620.0F;  // ~100M, for measuring small capacitance
double resistorValue = 0; 
unsigned long startTime;
unsigned long elapsedTime;
float microFarads;                // floating point variable to preserve precision, make calculations
float nanoFarads;
int reading = 0;
int delay_display1 = 3000;
int delay_display2 = 5000;
int cap_add = 70;                 // additional circuit capacitacne in pF
                                 
//initiallize the Lcd
LiquidCrystal_I2C lcd(0x27,16,2);  //sometimes the adress is not 0x3f. Change to 0x27 if it doesn't work. 

void setup() {

  lcd.begin();                  // lcd communication for output display
  lcd.backlight();              // make the lcd light On  
  lcd.print("Arduino inside");
  lcd.setCursor(0, 1);
  lcd.print("LC-meter");    
  pinMode(pin_mode_LM, INPUT); // sets the digital pin as input
  pinMode(pin_mode_LS, INPUT); // sets the digital pin as input
  pinMode(pin_mode_CL, INPUT); // sets the digital pin as input
  pinMode(pin_mode_CS, INPUT); // sets the digital pin as input
  delay(300);
  digitalWrite(pin_mode_LM, LOW);  // sets the digital pin off
  digitalWrite(pin_mode_LS, LOW);  // sets the digital pin off
  digitalWrite(pin_mode_CL, LOW);  // sets the digital pin off
  digitalWrite(pin_mode_CS, LOW);  // sets the digital pin off
  Serial.begin(9600);          // initialize serial transmission
  delay(500);
  Serial.println("LC-meter powered by Arduino");
  delay(delay_display1);
  lcd.clear();
}

void loop() {
 mode = digitalRead(pin_mode_LM)*1+digitalRead(pin_mode_LS)*2+digitalRead(pin_mode_CL)*4+digitalRead(pin_mode_CS)*8;
        switch (mode) {
            case 0:
                measure_IR_freq();
                break;          
            case 1:
                measure_freq();
                break;
            case 2:
                measure_inductance();
                break;
            case 4:
                resistorValue = resistorValue1;
                measure_capacitance();
                break;
            case 8:
                resistorValue = resistorValue2;
                measure_capacitance();
                break;                
          default:
                Serial.println("Error");
                lcd.setCursor(0,0);           
                lcd.print("Error           ");
                lcd.setCursor(0,1);           
                lcd.print("                ");
                break;
        }      
}

void measure_IR_freq() {
  
  flag_complete = true;
  start = 0;
  now = 0;
  i = 0;
  attachInterrupt(IR_PIN-2, Rec_Interrupt, CHANGE);  // Call the function, whenever
                                              // change in pulse is detected.
  Serial.println("Press the button");
  lcd.setCursor(0,0);           
  lcd.print("IR frequency    ");
  lcd.setCursor(0,1);
  lcd.print("Press the button");
  while (1) {
    if (flag_complete == 0) {
      
      /* Serial.print("Interrupt "); */
 
      for (i = 1; i < 3; i++) {
        if (i % 2 == 0)
        {
          Serial.print("LOW ");
          Serial.println(" microseconds");
        }

        else
        {
          Serial.print("HIGH ");
          Serial.print(capture[i]);
          Serial.println(" microseconds");
        }

        flag_complete = true;
      }
      for (int x = 1; x < 3; x++)
      {
        add += capture[x];  // Adding High Time and Low Time.
      }

      Serial.print("Period ");
      Serial.println(add);
      Serial.print("Frequency ");
      Serial.print((float)1000 / add, 5);
      Serial.println(" kHz");
      Serial.println(" ");
      lcd.setCursor(0,1);           
      lcd.print((float)1000 / add, 5);
      lcd.setCursor(7,1);
      lcd.print(" kHz     ");
      delay(2000);
    }

    add = 0;  // clear
    return;
  }
}

void measure_freq() {

 FreqCounter::f_comp= 10;                          // Set compensation to 12
 FreqCounter::start(1000);                         // Start counting with gatetime of 1s
 while (FreqCounter::f_ready == 0)                 // wait until counter ready
 frequency = float (FreqCounter::f_freq) /1000;    // in KHz
 serialPrint_freq();
 lcdPrint_freq();
 
}

void measure_inductance() {
 FreqCounter::f_comp= 10;             // Set compensation to 12
 FreqCounter::start(1000);            // Start counting with gatetime of 1s
 while (FreqCounter::f_ready == 0)    // wait until counter ready
 
 frq=FreqCounter::f_freq*prescaler/10;            // frequency/10 Hz
 frequency = frq /100;                // in KHz

 if (FreqCounter::f_freq < 5) inductance = 0; else inductance = 1e4/(frq*frq*cap*4*pi*pi); // inductance in uH using L = 1/(f^2*C*4*pi^2)
 inductance -= correction_inductance;
 inductance = max(0, inductance);
 serialPrint_inductance();
 lcdPrint_inductance();
}

void measure_capacitance() {
  
   pinMode(chargePin, OUTPUT);              // set chargePin to output

  // discharge the capacitor  
  digitalWrite(chargePin, LOW);             // set charge pin to  LOW 
  pinMode(dischargePin, OUTPUT);            // set discharge pin to output 
  digitalWrite(dischargePin, LOW);          // set discharge pin LOW 
  while(analogRead(analogPin) > 0) {        // wait until capacitor is completely discharged
  }
  delay(1000);
  pinMode(dischargePin, INPUT);             // set discharge pin back to input
   
   lcd.setCursor(0,0);           
   lcd.print("Capacitance     ");
   delay(delay_display1);
   lcd.setCursor(0,0);
   if (mode == 4) lcd.print("Large mode      ");
   if (mode == 8) lcd.print("Small mode      ");
   delay(delay_display1);
   lcd.setCursor(0,0);
   lcd.print("Measuring...  ");
   lcd.setCursor(0,1);
 
   startTime = millis();        
   digitalWrite(chargePin, HIGH);            // set chargePin HIGH and capacitor charging
   
   while(analogRead(analogPin) < 648) {      // 647 is 63.2% of 1023, which corresponds to full-scale voltage 
   }  

  elapsedTime= millis() - startTime;
 
  if (mode == 4 & elapsedTime < 3 & elapsedTime !=0) { 
      
      Serial.println("Switch to small mode");         // print units and carriage return
      lcd.setCursor(0,1);
      lcd.print("Switch to small mode");
      delay(delay_display1);
      return;
  }

  delay(1000);
  
  new_mode = digitalRead(pin_mode_LM)*1+digitalRead(pin_mode_LS)*2+digitalRead(pin_mode_CL)*4+digitalRead(pin_mode_CS)*8;
  if (mode == new_mode) {
     // convert milliseconds to seconds ( 10^-3 ) and Farads to microFarads ( 10^6 ),  net 10^3 (1000)  
      microFarads = ((float)elapsedTime / resistorValue) * 1000;   
      Serial.print(elapsedTime);       // print the value to serial port
      Serial.print(" mS    ");         // print units and carriage return
    
      if (microFarads >= 0.1) {
        Serial.print((float)microFarads);       // print the value to serial port
        Serial.println(" microFarads");         // print units and carriage return
        lcd.setCursor(0,1);
        lcd.print(microFarads);
        lcd.print(" uF           ");
        if (elapsedTime <= 5000) delay(delay_display1); else delay(delay_display2);
      }
      else {
        if (microFarads <= 0.01) {
    
        Serial.print(max(0,(float)microFarads*1000000-cap_add));       
        Serial.println(" picoFarads");         
        lcd.setCursor(0,1);
        reading = max(int(microFarads*1000000)-cap_add,0);
        lcd.print(reading);
        if (reading != 0) lcd.print(" pF           "); else lcd.print("              ");
        if (elapsedTime <= 5000) delay(delay_display1); else delay(delay_display2);
        }
        else { nanoFarads = microFarads * 1000.0;      // multiply by 1000 to convert to nanoFarads (10^-9 Farads)
        Serial.print((float)nanoFarads);         // print the value to serial port
        Serial.println(" nanoFarads");          // print units and carriage return
        lcd.setCursor(0,1);
        lcd.print(microFarads*1000);
        lcd.print(" nF           ");
        if (elapsedTime <= 5000) delay(delay_display1); else delay(delay_display2);
        }
      }
  }

}
       
void Rec_Interrupt() {
 
  now = micros();                 // Capture current time.
  capture[i++] = now - start;     // Subtract Current time and Previous time.
  start = now;                    // Previous time is equal to current time.
  if (i >= 3)                     // If count is equal to or greater than 3
  {
    detachInterrupt(IR_PIN-2);    // disable Interrupt 0
    flag_complete = false;        // clear flag.
  }
}

void serialPrint_freq() {

        Serial.print("\tfrequency KHz: ");
        Serial.println( frequency,3 );

}

void lcdPrint_freq() {
  
              lcd.clear();
              lcd.setCursor(0,0);           
              lcd.print("FREQUENCY       ");
              lcd.setCursor(5 , 1);
              lcd.print (frequency,3);
              lcd.print ("KHz");
              delay(500);

}

void serialPrint_inductance() {

        Serial.print("\tfrequency KHz:");
        Serial.print( frequency,3 );
        Serial.print("inductance uH:");
        Serial.println( inductance );

}

void lcdPrint_inductance() {
  
              inductance_units = "uH";
              lcd.clear();
              lcd.setCursor(0,0);           
              lcd.print("INDUCTANCE      ");
              lcd.setCursor(5 , 1);
              if (inductance > 1e6 ) {
                inductance_units = "H";
                inductance /= 1e6; 
              }
              if (inductance > 1e3 & inductance <= 1e6) {
                inductance_units = "mH";
                inductance /= 1e3; 
              }
              else ;
              
              if (inductance > 100 )
              lcd.print (round(inductance));
              else lcd.print (inductance,2);

              if (inductance == 0 ) inductance_units = "";
              
              lcd.print (inductance_units );
              delay(500);

}

// End of sketch
  ```
  

   The schematics and the sketch is  a compilation from various free sources one can find on the Internet and the original circuit and code.

   The sketch was tested with Arduino UNO R3.
   
   The LC-meter can also measure IR remote control modulation frequency and frequency of the digital signal compatible with 5V CMOS logic.

   Full functionality: IR remote rontrol tester, frequency measurement, inductance meter, capacitance meter with mode selection. 

   ### IR Remote Control Tester
   
   It is a rather toy functionality based on:
   
   https://www.instructables.com/id/Analyze-Any-IR-Protocol-With-Just-You-Arduino-Boar/
   
   SAMSUNG DVD-Karaoke IR Remote Control:
       
  ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/5.jpg "35.7142 kHz")


   ### Frequency measurement
   
   Tested in the range from 10Hz to 2MHz with the digital signal compatible with 5V CMOS logic.
   You can measure both internal frequency of the tank divided by prescaler value of 16 when external inductor is connected and external frequency.
   
   Based on:
   
   http://interface.khm.de/index.php/lab/interfaces-advanced/arduino-frequency-counter-library/
   
   
   Some pictures here:
   
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/3.jpg "0.440 kHz")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/6.jpg "2000 kHz")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/7.jpg "0.010 kHz")
   
   ###  Inductance meter
   
   Accuracy is ±3% in the  range  0.5uH - 10mH or better. 
   
   Based on:
   
   https://0jihad0.livejournal.com/2081.html
   
   https://www.radiolocman.com/shem/schematics.html?di=162628
   
   
   Colpitts oscillator was not used because it did not prove to be accurate enough in the required range of frequencies of the tank.
   Some frequncies were  200-300% different from theoretical
   
   ![alt text](https://wikimedia.org/api/rest_v1/media/math/render/svg/a9e393a8a5b2bc2aab57b5082c84a628d41069ce "Theoretical frequency of Colpitts oscillator")
 
   The oscillator I used proved to be  quite accurate in this respect. It provides all required frequencies almost as accurate as theoretical
   
  ![alt text](https://wikimedia.org/api/rest_v1/media/math/render/svg/99971532401f7f6241b673c877f346889bce39a7 "Tank frequency")
  
   
   Various "calibrated" inductors:
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/1.jpg "5.20mH")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/12.jpg "1.00mH")
   
   
   1uH and 2uH inductors connected in parallel (1/0.6666... = 1/1 + 1/2):
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/15.jpg "0.67uH")
   
   Just several rounds of wire:
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/16.jpg "0.46uH")
   
   ### Capacitance meter

   Tested with capacitances from 30 pF to 10000uF.
   There are two measurement modes:
   

        
   #### Large mode
   
       0.47uF - 10000uF and bigger.
       
   #### Small mode 
   
       0pF - 0.47uF
  
   Based on:
   
   https://www.arduino.cc/en/Tutorial/CapacitanceMeter
   
   
   In Large Mode, the capacitor discharges through 10k resistor. In small mode, the capacitor discharges through 87M resistor.
   The measurement may take from several seconds to several minutes to complete.
     
   Some pictures here:
   

   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/10.jpg "4700 uF")
    
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/4.jpg "0.10 uF")
    
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/23.jpg "200 pF")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/24.jpg "22 pF")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/25.jpg "10 pF")
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/26.jpg "0 pF")  
   
   ### Mode selection
   
   An original circuit and code. As simple as possible!
   
   2P5T rotary switch SW1 and SW2 are used for switching between measurement modes.
   
   Switches | Mode
   ---------|------
   SW1 position 1 | IR remote control tester
   SW1 position 2, SW2 closed | internal frequency measurement
   SW1 position 2, SW2 open | external frequency measurement
   SW1 position 3 | inductance measurement
   SW1 position 4 | capacitance measurement, large mode 
   SW1 position 5 | capacitance measurement, small mode
   
   ### Power
   
   The LC-meter is powered from  5V DC USB source of Arduino board. Current should not exceed 87 mA. The most current is consumed by the LCD.
   
   ### Calibration
   
   Calibration can be  performed programmatically using Arduino IDE.
   You may want to change the following values in the Arduino sketch, compile and upload the modified sketch to the Arduino board.
   
   https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/LC_meter.ino
   
   ##### Inductance meter calibration
   
   ```
   double cap = 1475e-12;                 // ~1500 pF, for measuring  inductance           
   float correction_inductance = 0.67;    // correction inductance, uH
   ```   
   
   ##### Capacitance meter calibration
   
   ```
   double resistorValue1 = 9620.0F;      // ~10k,  for measuring large capacitance
   double resistorValue2 = 87940620.0F;  // ~100M, for measuring small capacitance
   int cap_add = 70;                    // additional circuit capacitacne in pF
   ```
   
   ### References
   
   https://0jihad0.livejournal.com/2081.html
 
   http://radiomaster.ru/shemi/uzli/gener.php
   
   https://www.arduino.cc/en/Tutorial/CapacitanceMeter
   
   http://interface.khm.de/index.php/lab/interfaces-advanced/arduino-frequency-counter-library/
   
   https://www.rugged-circuits.com/10-ways-to-destroy-an-arduino
   
   https://www.instructables.com/id/Analyze-Any-IR-Protocol-With-Just-You-Arduino-Boar/
   
   https://github.com/johnrickman/LiquidCrystal_I2C
   
   
   ### Credits
    
   Thanks to all authors and  contributors.
   
   
   ![alt text](https://github.com/alpop/Arduino-UNO-R3-based-LC-meter/blob/master/pictures/13.jpg "LC_meter")
 
   
