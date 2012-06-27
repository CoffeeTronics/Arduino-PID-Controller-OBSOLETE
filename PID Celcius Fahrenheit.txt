#include <MenuBackend.h>    //MenuBackend library - copyright by Alexander Brevig
#include <LiquidCrystal.h>  //this library is modified by Lady Ada to support I2C
#include <Wire.h>
#include <Adafruit_MAX31855.h>

uint8_t degree[8] = {0xc,0x12,0xc,0x0,0x0,0x0,0x0};

const int buttonPinLeft = 7;      // pin for the Up button
const int buttonPinRight = 8;    // pin for the Down button
const int buttonPinEsc = 9;     // pin for the Esc button
const int buttonPinEnter = 10;   // pin for the Enter button

int lastButtonPushed = 0;

int lastButtonEnterState = LOW;   // the previous reading from the Enter input pin
int lastButtonEscState = LOW;   // the previous reading from the Esc input pin
int lastButtonLeftState = LOW;   // the previous reading from the Left input pin
int lastButtonRightState = LOW;   // the previous reading from the Right input pin


long lastEnterDebounceTime = 0;  // the last time the output pin was toggled
long lastEscDebounceTime = 0;  // the last time the output pin was toggled
long lastLeftDebounceTime = 0;  // the last time the output pin was toggled
long lastRightDebounceTime = 0;  // the last time the output pin was toggled
long debounceDelay = 500;    // the debounce time

float SPWhole = 0.0;
float SPDecimal = 0.00;
float SetPoint = 200.00;

int tempScale = 0;
int currentScale = 1;
int pinEnterCheck;

int adjustP = 0;
int adjustI = 0;
int adjustD = 0;
int changeP = 0;
int changeI = 0;
int changeD = 0;

unsigned int printP = 0;
unsigned int printI = 0;
unsigned int printD = 0;

unsigned int pinRightCheck = 0;
unsigned int pinLeftCheck = 0;

int Kp = 0;
int Ki = 0;
int Kd = 0;

LiquidCrystal lcd(0); //Default address for I2C LCD

//Menu variables
MenuBackend menu = MenuBackend(menuUsed,menuChanged);
//initialize menuitems
    MenuItem menu1Item1 = MenuItem("SetPointWhole");
      MenuItem menuItem1SubItem1 = MenuItem("190");
      MenuItem menuItem1SubItem2 = MenuItem("191");
      MenuItem menuItem1SubItem3 = MenuItem("192");
      MenuItem menuItem1SubItem4 = MenuItem("193");
      MenuItem menuItem1SubItem5 = MenuItem("194");
      MenuItem menuItem1SubItem6 = MenuItem("195");
      MenuItem menuItem1SubItem7 = MenuItem("196");
      MenuItem menuItem1SubItem8 = MenuItem("197");
      MenuItem menuItem1SubItem9 = MenuItem("198");
      MenuItem menuItem1SubItem10 = MenuItem("199");
      MenuItem menuItem1SubItem11 = MenuItem("200");
      MenuItem menuItem1SubItem12 = MenuItem("201");
      MenuItem menuItem1SubItem13 = MenuItem("202");
      MenuItem menuItem1SubItem14 = MenuItem("203");
      MenuItem menuItem1SubItem15 = MenuItem("204");
      MenuItem menuItem1SubItem16 = MenuItem("205");
      MenuItem menuItem1SubItem17 = MenuItem("206");
      MenuItem menuItem1SubItem18 = MenuItem("207"); 
  MenuItem menu1Item2 = MenuItem("SetPointDecimal");
      MenuItem menuItem2SubItem1 = MenuItem(".00");
      MenuItem menuItem2SubItem2 = MenuItem(".25");
      MenuItem menuItem2SubItem3 = MenuItem(".50");
      MenuItem menuItem2SubItem4 = MenuItem(".75");
  MenuItem menu1Item3 = MenuItem("Temp Scale");
      MenuItem menuItem3SubItem1 = MenuItem("Celcius");    
      MenuItem menuItem3SubItem2 = MenuItem("Fahrenheit"); 
  MenuItem menu1Item4 = MenuItem("Tuning");
      MenuItem menuItem4SubItem1 = MenuItem("Set P");
        MenuItem menuItem4SubItem1SubItem1 = ("P +/-");
      MenuItem menuItem4SubItem2 = MenuItem("Set I"); 
        MenuItem menuItem4SubItem2SubItem1 = ("I +/-");
      MenuItem menuItem4SubItem3 = MenuItem("Set D"); 
        MenuItem menuItem4SubItem3SubItem1 = ("D +/-");

void setup() {
    Serial.begin(9600);  
    lcd.begin(16, 2);
    lcd.setBacklight(HIGH);
    delay (500);    
   // lcd.createChar(0, degree);
    lcd.setCursor(0, 0);
    
    
    // Initial splash screen
    delay(500);
    lcd.print("Arduino ");
    delay(500);
    lcd.print("P");
    delay(500);
    lcd.print("I");
    delay(500);
    lcd.print("D");
    delay(500);
    lcd.setCursor(2, 1);
    lcd.print("for espresso");
    delay(3000);
    lcd.clear();      
            
    pinMode(buttonPinLeft, INPUT);
    pinMode(buttonPinRight, INPUT);
    pinMode(buttonPinEnter, INPUT);
    pinMode(buttonPinEsc, INPUT);
    
     //configure menu
    
    menu.getRoot().add(menu1Item1);
  menu1Item1.addRight(menu1Item2).addRight(menu1Item3).addRight(menu1Item4);
  menu1Item1.add(menuItem1SubItem1).addRight(menuItem1SubItem2).addRight(menuItem1SubItem3).addRight(menuItem1SubItem4).addRight(menuItem1SubItem5).addRight(menuItem1SubItem6).addRight(menuItem1SubItem7).addRight(menuItem1SubItem8).addRight(menuItem1SubItem9).addRight(menuItem1SubItem10).addRight(menuItem1SubItem11).addRight(menuItem1SubItem12).addRight(menuItem1SubItem13).addRight(menuItem1SubItem14).addRight(menuItem1SubItem15).addRight(menuItem1SubItem16).addRight(menuItem1SubItem17).addRight(menuItem1SubItem18);
  menu1Item2.add(menuItem2SubItem1).addRight(menuItem2SubItem2).addRight(menuItem2SubItem3).addRight(menuItem2SubItem4);
  menu1Item3.add(menuItem3SubItem1).addRight(menuItem3SubItem2);
  menu1Item4.add(menuItem4SubItem1).addRight(menuItem4SubItem2).addRight(menuItem4SubItem3);
  menuItem4SubItem1.addAfter(menuItem4SubItem1SubItem1);
  menuItem4SubItem2.addAfter(menuItem4SubItem2SubItem1);
  menuItem4SubItem3.addAfter(menuItem4SubItem3SubItem1);
    
    
    menu.toRoot();
    lcd.setCursor(0, 0);
    lcd.print("S:");
    lcd.setCursor(2, 0);
    lcd.print(SetPoint);
    //lcd.print(0);
   
}

void loop() {
  
   //lcd.clear();
    //lcd.setCursor(8, 0);
    // Printing the PV, or
   // lcd.print("P:");      // Process Variable. This is
   //lcd.setCursor(10, 0); // the measured value of temp.
   
    readButtons();    //I split button reading and navigation in two procedures because
    navigateMenus();  //in some situations I want to use the button for other purpose (eg. to change some settings)
    changeTuning();
    tempConversion();
} //end loop

void menuChanged(MenuChangeEvent changed) {
    MenuItem newMenuItem = changed.to;
    char* newMenuItemName = const_cast<char *>(newMenuItem.getName());
    int newSetting = 0;
    
    //get the destination menu
     lcd.setCursor(0, 1);
     
   //set the start position for lcd printing to the second row
    if(newMenuItemName == menu.getRoot().getName()){
      lcd.print("Menu            ");
      changeP = 0;
      changeI = 0;
      changeD = 0;
      tempScale = 0;
    } else if(newMenuItemName == "SetPointWhole"){
      lcd.print("Temp Whole      ");   
      tempScale = 0;
    } else if(newMenuItemName == "SetPointDecimal"){
      lcd.print("Temp Decimal    ");
      tempScale = 0;  
    } else if(newMenuItemName == "190"){  
      SPWhole=190;
      lcd.print("190            ");
      tempScale = 0;
    } else if(newMenuItemName == "191"){  
      SPWhole=191;
      lcd.print("191            ");
      tempScale = 0;
    } else if(newMenuItemName == "192"){  
      SPWhole=192;
      lcd.print("192            ");
      tempScale = 0;
    } else if(newMenuItemName == "193"){  
      SPWhole=193;
      lcd.print("193            ");
      tempScale = 0;
    } else if(newMenuItemName == "194"){  
      SPWhole=194;
      lcd.print("194            ");
      tempScale = 0;
    } else if(newMenuItemName == "195"){  
      SPWhole=195;
      lcd.print("195            ");
      tempScale = 0;
    } else if(newMenuItemName == "196"){  
      SPWhole=196;
      lcd.print("196            ");
      tempScale = 0;
    } else if(newMenuItemName == "197"){  
      SPWhole=197;
      lcd.print("197            ");
      tempScale = 0;
    } else if(newMenuItemName == "198"){  
      SPWhole=198;
      lcd.print("198            ");
      tempScale = 0;
    } else if(newMenuItemName == "199"){  
      SPWhole=199;
      lcd.print("199            ");
      tempScale = 0;
    } else if(newMenuItemName == "200"){  
      SPWhole=200;
      lcd.print("200            ");
      tempScale = 0;
    } else if(newMenuItemName == "201"){  
      SPWhole=201;
      lcd.print("201            ");
      tempScale = 0;
    } else if(newMenuItemName == "202"){  
      SPWhole=202;
      lcd.print("202            ");
      tempScale = 0;
    } else if(newMenuItemName == "203"){  
      SPWhole=203;
      lcd.print("203            ");
      tempScale = 0;
    } else if(newMenuItemName == "204"){  
      SPWhole=204;
      lcd.print("204            ");
      tempScale = 0;
    } else if(newMenuItemName == "205"){  
      SPWhole=205;
      lcd.print("205            ");
      tempScale = 0;
    } else if(newMenuItemName == "206"){  
      SPWhole=206;
      lcd.print("206            ");
      tempScale = 0;
    } else if(newMenuItemName == "207"){  
      SPWhole=207;
      lcd.print("207            ");
      tempScale = 0;
    } else if(newMenuItemName == ".25"){  
      SPDecimal=0.25;
      lcd.print(".25            ");
    } else if(newMenuItemName == ".50"){  
      SPDecimal=0.50;
      lcd.print(".50            ");
      tempScale = 0;
    } else if(newMenuItemName == ".75"){  
      SPDecimal=0.75;
      lcd.print(".75            ");
      tempScale = 0;
    } else if(newMenuItemName == ".00"){  
      SPDecimal=0.00;
      lcd.print(".00            ");
      tempScale = 0;
    } else if(newMenuItemName == "Temp Scale"){
      lcd.print("Temp Scale");
      lcd.print("   ");
      tempScale = 0;
    } else if(newMenuItemName == "Celcius"){
      lcd.print("Celcius");
      lcd.print("   ");
      tempScale = 2;
    } else if(newMenuItemName == "Fahrenheit"){
      lcd.print("Fahrenheit");  
      tempScale = 1;
    }else if(newMenuItemName == "Tuning"){
      lcd.print("Tuning");
      lcd.print("    ");
      tempScale = 0;
    } else if(newMenuItemName == "Set P"){
      lcd.print("Set P           ");  
      tempScale = 0;
    } else if(newMenuItemName == "Set I"){
      lcd.print("Set I           "); 
     tempScale = 0; 
    } else if(newMenuItemName == "Set D"){
      lcd.print("Set D           ");  
      tempScale = 0;
    }else if(newMenuItemName == "P +/-"){
      lcd.print("P +/-  ");
      lcd.print("P=");
      lcd.print(printP);
      changeP = 1;
      tempScale = 0;
    }else if(newMenuItemName == "I +/-"){
      lcd.print("I +/-  ");
      lcd.print("I=");
      lcd.print(printI);
      changeI = 1;
      tempScale = 0;
    }else if(newMenuItemName == "D +/-"){
      lcd.print("D +/-  ");
      lcd.print("D=");
      lcd.print(printD);
      changeD = 1;
      tempScale = 0;
    }
    else
   {
     tempScale = 0;
   }
           
  //  }else if ((newMenuItemName == "P +/-") && (buttonPinRight == HIGH)) {
    //  changeP = 1;
    }

    
  int changeTuning()
  {
       pinRightCheck = digitalRead(buttonPinRight);
    if ((changeP == 1) && (pinRightCheck == HIGH)) 
    {
      Kp +=1;
      delay(100);
      printP = Kp;
     // return Kp;
      lcd.setCursor(7,1);
      lcd.print("P=");
      lcd.print(printP);
    }
   pinLeftCheck = digitalRead(buttonPinLeft);
    //adjustP = changeP;
    if ((changeP == 1) && (pinLeftCheck == HIGH)) 
    {
      Kp -=1;
      delay(100);
      printP = Kp;
     // return Kp;
      lcd.setCursor(7,1);
      lcd.print("P=");
      lcd.print(printP);
    }
 pinRightCheck = digitalRead(buttonPinRight);
      if ((changeI == 1) && (pinRightCheck == HIGH)) {
      Ki +=1;
      delay(100);
      printI = Ki;
      lcd.setCursor(7,1);
      lcd.print("I=");
      lcd.print(printI);
    }
   pinLeftCheck = digitalRead(buttonPinLeft);
      if ((changeI == 1) && (pinLeftCheck == HIGH)) {
      Ki -=1;
      delay(100);
      printI = Ki;
      lcd.setCursor(7,1);
      lcd.print("I=");
      lcd.print(printI);
	  }
 pinRightCheck = digitalRead(buttonPinRight);
      if ((changeD == 1) && (pinRightCheck == HIGH)) {
      Kd +=1;
      delay(100);
      printD = Kd;
      lcd.setCursor(7,1);
      lcd.print("D=");
      lcd.print(printD);
    }
   pinLeftCheck = digitalRead(buttonPinLeft);
      if ((changeD == 1) && (pinLeftCheck == HIGH)) {
      Kd -=1;
      delay(100);
      printD = Kd;
      lcd.setCursor(7,1);
      lcd.print("D=");
      lcd.print(printD);
	  }
  }
    
  void menuUsed(MenuUseEvent used) {
    SetPoint = SPWhole + SPDecimal;
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("S:");
    lcd.setCursor(2,0);
    lcd.print(SetPoint);
    menu.toRoot();
}

void readButtons() {
    //read buttons status
    int reading;
    int buttonEnterState = LOW;    // the current reading from the Enter input pin
    int buttonEscState = LOW;      // the current reading from the input pin
    int buttonLeftState = LOW;     // the current reading from the input pin
    int buttonRightState = LOW;    // the current reading from the input pin
    //Enter button
    // read the state of the switch into a local variable:
    reading = digitalRead(buttonPinEnter);

    // check to see if you just pressed the enter button
    // (i.e. the input went from LOW to HIGH), and you've waited
    // long enough since the last press to ignore any noise:
    // If the switch changed, due to noise or pressing:
    if (reading != lastButtonEnterState) {
        // reset the debouncing timer
        lastEnterDebounceTime = millis();
    }

    if ((millis() - lastEnterDebounceTime) > debounceDelay) {
        // whatever the reading is at, it's been there for longer
        // than the debounce delay, so take it as the actual current state:
        buttonEnterState = reading;
        lastEnterDebounceTime = millis();
    }

    // save the reading. Next time through the loop,
    // it'll be the lastButtonState:
    lastButtonEnterState = reading;


    //Esc button
    // read the state of the switch into a local variable:
    reading = digitalRead(buttonPinEsc);

    // check to see if you just pressed the Down button
    // (i.e. the input went from LOW to HIGH), and you've waited
    // long enough since the last press to ignore any noise:
    // If the switch changed, due to noise or pressing:
    if (reading != lastButtonEscState) {
        // reset the debouncing timer
        lastEscDebounceTime = millis();
    }

    if ((millis() - lastEscDebounceTime) > debounceDelay) {
        // whatever the reading is at, it's been there for longer
        // than the debounce delay, so take it as the actual current state:
        buttonEscState = reading;
        lastEscDebounceTime = millis();
    }

    // save the reading. Next time through the loop,
    // it'll be the lastButtonState:
    lastButtonEscState = reading;


    //Down button
    // read the state of the switch into a local variable:
    reading = digitalRead(buttonPinRight);

    // check to see if you just pressed the Down button
    // (i.e. the input went from LOW to HIGH), and you've waited
    // long enough since the last press to ignore any noise:
    // If the switch changed, due to noise or pressing:
    if (reading != lastButtonRightState) {
        // reset the debouncing timer
        lastRightDebounceTime = millis();
    }

    if ((millis() - lastRightDebounceTime) > debounceDelay) {
        // whatever the reading is at, it's been there for longer
        // than the debounce delay, so take it as the actual current state:
        buttonRightState = reading;
        lastRightDebounceTime = millis();
    }

    // save the reading. Next time through the loop,
    // it'll be the lastButtonState:
    lastButtonRightState = reading;


    //Up button
    // read the state of the switch into a local variable:
    reading = digitalRead(buttonPinLeft);

    // check to see if you just pressed the Down button
    // (i.e. the input went from LOW to HIGH), and you've waited
    // long enough since the last press to ignore any noise:
    // If the switch changed, due to noise or pressing:
    if (reading != lastButtonLeftState) {
        // reset the debouncing timer
        lastLeftDebounceTime = millis();
    }

    if ((millis() - lastLeftDebounceTime) > debounceDelay) {
        // whatever the reading is at, it's been there for longer
        // than the debounce delay, so take it as the actual current state:
        buttonLeftState = reading;
        lastLeftDebounceTime = millis();
    }

    // save the reading. Next time through the loop,
    // it'll be the lastButtonState:
    lastButtonLeftState = reading;

    //records which button has been pressed
    if (buttonEnterState == HIGH) {
        lastButtonPushed = buttonPinEnter;

    } else if (buttonEscState == HIGH) {
        lastButtonPushed = buttonPinEsc;

    } else if (buttonRightState == HIGH) {
        lastButtonPushed = buttonPinRight;

    } else if (buttonLeftState == HIGH) {
        lastButtonPushed = buttonPinLeft;

    } else {
        lastButtonPushed = 0;
    }
}

void navigateMenus() {
    MenuItem currentMenu = menu.getCurrent();

    switch (lastButtonPushed) {
    case buttonPinEnter:
        if (! (currentMenu.moveDown())) { // if the current menu has a child and has been pressed enter then menu navigate to item below
            menu.use();
        } else {                          // otherwise, if menu has no child and has been pressed enter the current menu is used
            menu.moveDown();
        }
        break;
    case buttonPinEsc:
        //menu.moveBack(); //go back up one branch
        menu.toRoot();
        //back to main
        break;
    case buttonPinRight:
        menu.moveRight();
        break;
    case buttonPinLeft:
        menu.moveLeft();
        break;
    }

    lastButtonPushed = 0;
    //reset the lastButtonPushed variable
}
    
  void printingSV() {
    //for printing the SetValue as the menu is being traversed
    SetPoint = SPWhole + SPDecimal;
    lcd.clear();
    lcd.print("S:");
   // intSetpoint = Setpoint * 1;
   lcd.print(SetPoint);
    lcd.print(0);
}   
     
   void tempConversion()
   {
     delay(250);
    pinEnterCheck = digitalRead(buttonPinEnter);
    
    if ((tempScale == 1)&&(currentScale == 1)&&(pinEnterCheck == HIGH))
    {
      lcd.setCursor(0,0);
      lcd.print("Already set to F");
      delay(500);
      lcd.clear();
      lcd.setCursor(0,0);
      lcd.print("S:");
      lcd.print(SetPoint);
      currentScale = 1;
       menu.toRoot();
    }
    else if ((tempScale == 2)&&(currentScale == 1)&&(pinEnterCheck == HIGH))
    {
      SetPoint = ((SetPoint - 32.0) * (5.0/9.0)); //Convert to celcius
      lcd.setCursor(0,0);
      lcd.print("S:");
      lcd.print(SetPoint);
      lcd.print("         ");
      lcd.setCursor(0,1);
      lcd.print("                ");
      currentScale = 2;
      delay(500);
      menu.toRoot();
    }
    else if ((tempScale == 2)&&(currentScale == 2)&&(pinEnterCheck == HIGH))
    {
      lcd.setCursor(0,0);
      lcd.print("Already set to C");
      delay(500);
      lcd.clear();
      lcd.setCursor(0,0);
      lcd.print("S:");
      lcd.print(SetPoint);
      currentScale = 2; //set currentscale to C
      menu.toRoot();
      
    }
    else if ((tempScale == 1)&&(currentScale == 2)&&(pinEnterCheck == HIGH))
   {
     SetPoint = ((SetPoint * (9.0/5.0)) + 32); //convert to F
     lcd.setCursor(0,0);
     lcd.print("S:");
     lcd.print(SetPoint);
     lcd.print("      ");
     currentScale = 1; //set current scale to F
     delay(500);
     menu.toRoot();
   }
   }   