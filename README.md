STARTUP(System.enableFeature(FEATURE_RETAINED_MEMORY)); // Enables backup RAM
STARTUP(WiFi.selectAntenna(ANT_EXTERNAL)); // choose external antenna
//STARTUP(WiFi.selectAntenna(ANT_AUTO)); //continually switches at high speed between antennas
SYSTEM_MODE(MANUAL); // Puts device in manual mode
SYSTEM_THREAD(ENABLED);

#include <string.h> 

const int PIN_LED = D7; // This is the LED that is already on your device.
const int PIN_BUTTON = A7; //this is where the water level switch plugged in

//Helper variables for the event payload
char buffer_open[10];
char buffer_close[10];
String strResult;

const char FW_VERSION[8] = "T_0.4.3";  //FW Version

const char pwd_id[13] = "94103ED40760"; //"94103ED40760"sean,"94103ED40E04" Lorenzo TODO: Read this from cloud and store in EEPROM
const char FIXTURE_NAME[8] = "Toilet2"; //TODO: Read this from cloud and store in EEPROM


int open_ts, close_ts; //Open and close timestamps
retained int ETA_Wake = 0;
retained int sync_t; //flag to tell if the RTC has been sync'ed or not

#define WAKEUP_RANGE 10; //Tolerance range to wake up for heartbeat in seconds
#define ONE_DAY 86400 //Number of seconds in a day

int lower_ETA;
int higher_ETA;
int wakeTime;

void setup() {
  // This part is mostly the same:
  pinMode(PIN_BUTTON, INPUT);
  pinMode(PIN_LED, OUTPUT); // Our on-board LED is output
  //Serial.begin(9600);
  //Serial.println("Starting...");
}


void loop() {
    delay(100);
    if (sync_t == 0)  //only run once to sync the RTC with cloud
    {
        tryConnect();
        Particle.syncTime();
        digitalWrite(PIN_LED, HIGH);
        delay(300);
        digitalWrite(PIN_LED, LOW);
        delay(200);
        sync_t = 1;
        //Serial.println("RTC sync'ed.");
        digitalWrite(PIN_BUTTON, LOW);
    }
    
    wakeTime = Time.now();
    lower_ETA = ETA_Wake - WAKEUP_RANGE;
    higher_ETA = ETA_Wake + WAKEUP_RANGE;

    if ((wakeTime > higher_ETA || wakeTime < lower_ETA)  && ETA_Wake != 0)      //check if it wakes up due to button or due to time
    {
        //due to button   
        if(digitalRead(PIN_BUTTON) == HIGH)   //check if the button/sensor is close(someone has pushed the button/flushed the toilet)
        {
            //Serial.print("button pressed!");
            open_ts = Time.now();  //mark down the starting time
            while(digitalRead(PIN_BUTTON) == HIGH) {
                delay(50); //wake up until the button.sensor is open
                //Serial.println("Still high...");
            }    
            
            close_ts = Time.now();  // mark down the ending time
            
            int gap = close_ts - open_ts;   //count the time between start and finish (sec)
            //Serial.print("Button Release. gap=");
            //Serial.println(gap);

            if(gap >= 3 && gap <= 10)   // If button is pressed between 3 and 10 secs, enter OTA mode  
            {
                //Serial.println("Button pressed for OTA mode. Keeping awake for 30 sec.");
                //Keep Awake for n secs
                tryConnect();
                flashLed(500, 60); //Keep Awake for 30 secs
                gotoSleep();
            }
            else 
            {
                //Serial.println("Publishing sendgtdata.");
                //publish "sendgtdata"
                tryConnect();
                itoa(open_ts, buffer_open, 10);
                itoa(close_ts, buffer_close, 10);
                
                //strResult = String::format("{'pwdid':'%s','ver':'%s','fxid':'%s','actions': {'open':'%s','close':'%s'}}"
                //    , pwd_id, FW_VERSION, FIXTURE_NAME, buffer_open, buffer_close);
                //delay(500);
                //Spark.publish("sendgtdata", strResult, 60, PRIVATE);
                
                strResult = String::format("{'home_id':'%s','events': [{'e':'o','t':'t','ts':%s},{'e':'c','t':'t','ts':%s}]}"  // because hot and cold swop, i changed "close_hot" to "close_cold"
                     , pwd_id, buffer_open,buffer_close);
                delay(500);
                Spark.publish("sendgtdata2", strResult, 60, PRIVATE);

                
                flashLed(100, 15);
                gotoSleep();
            }
        }
    }
    else   // publish to show the photon still alive when no acticity for a whole day
    {
        //due to time 
        tryConnect();
        Spark.publish("heartbeat", "Still alive, no activity all day", 60, PRIVATE);
        delay(3500);
    }  
    
    gotoSleep();
}    
 
//Sends Photon to deep sleep mode
void gotoSleep(){
    //Serial.println("Going to Zzzzz...");
    flashLed(50, 60);
    ETA_Wake = Time.now() + ONE_DAY;  //wake up once every day if no activity for a whole day.
    System.sleep(SLEEP_MODE_DEEP, ONE_DAY);
}

//Flashes LED for period (in ms) and number of times
void flashLed(int period, int times){
    for(int i = 0; i < times; i++){
        digitalWrite(PIN_LED, HIGH);    
        delay(period);
        digitalWrite(PIN_LED, LOW);
        delay(period);
    }
}

//Try to connect to Particle cloud
void tryConnect()
{
    Particle.connect();
    if (!waitFor(Particle.connected, 40000)) 
    { // If we do not connect to Particle Clould in 5 mins run the following sleep code. 
        gotoSleep();
    }
}
