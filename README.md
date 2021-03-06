 #include<iostream>
        #include<fcntl.h>
        #include<sys/ioctl.h>
        #include<unistd.h>

        #include<linux/i2c-dev.h>
        #ifndef I2C_FUNC_I2C
        #include  <linux/i2c.h> 
        #endif
        using namespace std;
        #define CLCK_ADD 0x68
        #define BUFFER_SIZE 19

        //DS3231 Registers with the address
        #define DS3231_SECONDS  0x00
        #define DS3231_MINUTES  0x01
        #define DS3231_HOURS  0x02
        #define DS3231_DAY 0x03
        #define DS3231_DATE 0x04
        #define DS3231_MONTH 0x05
        #define DS3231_YEAR 0x06
        #define DS3231_ALARM1_SECONDS 0x07
        #define DS3231_ALARM1_MINUTES 0x08
        #define DS3231_ALARM1_HOURS 0x09
        #define DS3231_ALARM1_DAY_DATE 0x0a
        #define DS3231_ALARM2_MINUTES 0x0b
        #define DS3231_ALARM2_HOURS 0x0c
        #define DS3231_ALARM2_DAY_DATE 0x0d
        #define DS3231_CONTROL 0x0e
        #define DS3231_CTL_STATUS 0x0f
        #define DS3231_AGING_OFFSET 0x10
        #define DS3231_TEMP_MSB 0x11
        #define DS3231_TEMP_LSB 0x12 

        //Converting Binary Coded Decimal to Decimal
        int binaryCDToDecimal(char b)
        {
            return (b/16*10) + (b%16);
        }
        //Converting Decimal to Binary Coded Decimal
        char decimalToBinaryCD(int d)
        {
            return ((d/10 * 16) + (d % 10));
        }
        
        int main()
        {
            //Reading and displaying  current to  RTC module time and date
            int file;
            char buf[BUFFER_SIZE];
            //Open the i2c-1 bus
            if ((file=open("/dev/i2c-1", O_RDWR)) < 0)
            {
                cout << "Failed to open bus\n";
                return 1;
            }
            //Connecting to the sensor using address 0x68
            if ((ioctl(file, I2C_SLAVE, 0x68)) < 0)
            {
                cout << "Failed to connect to the sensor\n";
                return 1;
            }
            char writeBuffer[1] = {0x00};
            //Initialize the read address
            if ((write(file, writeBuffer, 1)) != 1)
            {
                cout << "Failed to reset  read address\n";
                return 1;
            }
            //Read the buffer
            if (read(file, buf, BUFFER_SIZE) != BUFFER_SIZE)
            {
                cout << "Failed to read in  buffer\n";
                return 1;
            }
            /* This Displays current RTC Time in HH:MM:SS format
            As the register stores the value in BCD format which is converted to decimal format to display */
            cout << "RTC Time in HH:MM:SS format is : " << binaryCDToDecimal(buf[2]) <<":"<< binaryCDToDecimal(buf[1]) << ":" << binaryCDToDecimal(buf[0]) << "\n";
            //Displays the current RTC Date in DD/MM/YY format
            cout << "RTC Date in DD/MM/YY format is : " << binaryCDToDecimal(buf[4]) <<"/"<< binaryCDToDecimal(buf[5]) << "/" << binaryCDToDecimal(buf[6]) << "\n";

            /*Read and display  current temperature
            Displays the RTC Temperature in ??C*/
            float temperature = buf[17] + ((buf[18]>>6)*0.25);//As a last 2 bit of the MSB is the decimal value, it is shifted right by 6 times and multiplied by 0.25 for resolution
            cout << "Temperature is " << temperature <<"??C\n";

            //Setting the current time & date to the RTC module.
            int writeSeconds = 25, writeMinutes = 59, writeHours = 15, writeDate = 04, writeMonth = 03, writeYear = 21;
            unsigned char buffer[2];
            //Set seconds
            buffer[0] = DS3231_SECONDS;//Address where the value is written
            buffer[1] = decimalToBinaryCD(writeSeconds);//To write into register, the decimal value is converted into BCD format
            write(file, buffer, 2);//For writing, the RTC requires address and the value to be written into address

            //Setting minutes
            buffer[0] = DS3231_MINUTES;
            buffer[1] = decimalToBinaryCD(writeMinutes) ;
            write(file, buffer, 2);

            //Setting hours 24 hour format
            buffer[0] = DS3231_HOURS;
            buffer[1] = decimalToBinaryCD(writeHours) ;
            write(file, buffer, 2);

            //Setting date
            buffer[0] = DS3231_DATE;
            buffer[1] = decimalToBinaryCD(writeDate) ;
            write(file, buffer, 2);

            //Setting Month
            buffer[0] = DS3231_MONTH;
            buffer[1] = decimalToBinaryCD(writeMonth) ;
            write(file, buffer, 2);

            //Setting Year
            buffer[0] = DS3231_YEAR;
            buffer[1] = decimalToBinaryCD(writeYear) ;
            write(file, buffer, 2);

            //Setting and read the two alarms
            //Alarm 1 is used to set the time on date
            int writeAlarmSeconds = 30, writeAlarmMinutes = 58, writeAlarmHours = 23, writeAlarmDate = 15;//(DT/DY) Flag is set as 0 to act as date
            //Set Seconds for ALarm 1
            buffer[0] = DS3231_ALARM1_SECONDS;
            buffer[1] = decimalToBinaryCD(writeAlarmSeconds) ;
            write(file, buffer, 2);

            //Setting Minutes for ALarm 1
            buffer[0] = DS3231_ALARM1_MINUTES;
            buffer[1] = decimalToBinaryCD(writeAlarmMinutes) ;
            write(file, buffer, 2);

            //Setting Hours for ALarm 1
            buffer[0] = DS3231_ALARM1_HOURS;
            buffer[1] = decimalToBinaryCD(writeAlarmHours) ;
            write(file, buffer, 2);

            //Setting Date for ALarm 1
            buffer[0] = DS3231_ALARM1_DAY_DATE;
            buffer[1] = decimalToBinaryCD(writeAlarmDate) ;
            write(file, buffer, 2);

            //Alarm 2 is used for setting the time on day
            writeAlarmMinutes = 55, writeAlarmHours = 23, writeAlarmDate = 45;//(DT/DY) Flag is set as 1 to act as day
            //Set Minutes for ALarm 2
            buffer[0] = DS3231_ALARM2_MINUTES;
            buffer[1] = decimalToBinaryCD(writeAlarmMinutes) ;
            write(file, buffer, 2);

            //Setting Hours for ALarm 2
            buffer[0] = DS3231_ALARM2_HOURS;
            buffer[1] = decimalToBinaryCD(writeAlarmHours) ;
            write(file, buffer, 2);

            //Setting Day for ALarm 2
            buffer[0] = DS3231_ALARM2_DAY_DATE;
            buffer[1] = decimalToBinaryCD(writeAlarmDate) ;
            write(file, buffer, 2);
            //Read and display the alarms
            write(file, writeBuffer, 1);//Initialze the buffer
            read(file, buf, BUFFER_SIZE);//Read the registers

            string weekDays[7]= {"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"};//Declare array to for day
            int day = binaryCDToDecimal(buf[13]) % 10; //Get the unit digit of the decimal value to display day mentioned in array
            //Display Alarm 1 for date
            cout << "RTC Alarm 1 for date " << binaryCDToDecimal(buf[10]) <<" is "<< binaryCDToDecimal(buf[9]) << ":" << binaryCDToDecimal(buf[8]) << ":" << binaryCDToDecimal(buf[7]) <<"\n";
            //Display Alarm 2 for day
            cout << "RTC Alarm 2 for every "<< weekDays[day] << " is " << binaryCDToDecimal(buf[12]) << ":" << binaryCDToDecimal(buf[11]) <<"\n";

            char writeControl = 0x1F;//A1IF is set to 1 is alarm register and timer register matches
            buffer[0] = DS3231_CONTROL;
            buffer[1] = writeControl;
            write(file, buffer, 2);
            write(file, writeBuffer, 1);
            read(file, buf, BUFFER_SIZE);
            
            //For Square Wave we set BSQW pin is set to 1 and INTCN Flag to 0
            writeControl = 0x58;
            buffer[0] = DS3231_CONTROL;
            buffer[1] = writeControl;
            write(file, buffer, 2);

            close(file);
            return 0;
        }
