***2/2024*** new saison starts and I will go on testing and optimizing the solution.

But at the moment it looks like I change to new solution for this zero grid and also possibility to only forward incoming minimal solar in bad weather situations from PS to grid (without stressing Delta 2 Max in low solar incoming weather situations)
by using a modified Smartplug so that permanentWatts (EEPROM) is not written so often and Smartplug uses default dynamicWatts instead.
So far looks promising... will go on investigating when weather gets better...
(so two solutions will be possible 😉)

***3/2024*** testing new solution using modified SmartPlug:

UseCases:
- Zero Grid
- SolarWatt PS map to AC Output
  
![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/0f9d0d68-73f0-46d3-a372-a6aba4a3f977)

***1) Good Weather condition***: Zero Grid, try to reach Zero grid supply
   
(image will follow when next good weather is coming)

***2) Bad Weather Condition***, forward all SolarIn from Powerstream directly to AC Output (dynamicWatts) and do not stress the D2M by minimal load/unload

![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/8f175de8-7d40-4bda-ad53-32dbd01f933d)


***Example: 1 good day followed by 2 bad weather days:*** with changing of UseCases

(Smartplug is going up to max of 2500W (his overvoltage protection in his app config) in Zero Grid mode (here was multiple usages of washing maschine), but PS Inv Output is only 600W because of the official limitation)

![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/2bc168f0-bd84-4b90-96da-53968166a294)

Keep on testing... ;-)

------------------------------

### Work in Progress: Ecoflow Powerstream's AC Output controlled from Home Assistant by using the ecoflow-powerstream-nodejs

For documentation reference for myself and maybe useful for others also (on your own risk ;-)…
Information on how I got this ecoflow-powerstream-nodejs (https://github.com/bogdancs92/ecoflow-powerstream-nodejs) solution running - some is Quick&Dirty and needs optimization but intention was to get it running and I can test further requirements ;-)


### First of all some notes:
-	My main intention, control Powerstream via Home Assistant (ideally directly in local network – but seems not possible in short term, so waiting for cloud solution when HA integration “ecoflow_cloud” is able to not only read Powerstream values but also control it), so following came up for me: 
-	I started also using iobroker and the provided script (https://forum.iobroker.net/topic/66743/ecoflow-connector-script-zur-dynamischen-leistungsanpassung) with interfacing to Home Assistant (where my Smartmeter values are, send relevant value via mqtt to iobroker for using in iobrober script and not the whole HA integration in iobroker because of performance reason and because I don’t need all the HA entities in iobroker, …)
I find the script and work very good, but in general/for the beginning I only want to control AC Output and maybe Power Supply Mode of Powerstream from my already running Home Assistant installation (where I have the Smartmeter Live Usage already available), therefore the script got to detailed for me (maybe haven’t spent enough time for understand all the options) over the last time and also I don’t want to maintain an iobrober installation in parallel.
But – the script and the work **Waly_de** spent into it is GREAT - maybe later I will continue using the script or whatever will popup in the near future ;-)

-	Then I found (was mentioned in the iobroker thread https://forum.iobroker.net/topic/66743/ecoflow-connector-script-zur-dynamischen-leistungsanpassung/245) this nodesjs solution (which was set up by @bogdancs92  based on the above iobroker script)

**- SO THANKS TO BOTH TO GET TO THIS STAGE ;-)**


My plan now was to use this nodejs solution in a docker container beside my already running Home Assistant docker container (but this is optional, can also be used without container directly within nodejs on OS level)



### For my first PoC I started with:

### **ecoflow-powerstream-nodejs**
* Cloned the repo from here (https://github.com/bogdancs92/ecoflow-powerstream-nodejs.git)
* Nodejs Powerstream Config change: (values for e.g. URL, QUERY, TOKEN are free to use but must correspond to the REST-URL which will be called later) 
    * Create/edit “.env”
```
NODE_ENV=development
KEY_MAIL=your_email_ecoflow_app
KEY_URL=cmd
KEY_PASSWORD=your_password_ecoflow_app
KEY_POWERSTREAM_SN=HW51xxxxxxxxxxxxx
KEY_QUERY_AC=ac_output_watt
KEY_QUERY_PRIO=power_supply_mode
TOKEN=my_token
TOKEN_VAL=my_secret_for_token
```
* (optional) Nodejs Powerstream Container with nodejs-server waiting for request to change AC output Watts or Power Supply Mode on Powerstream
*  _Without container: “node server.js”_ can be used also without a container when nodejs is installed in OS, container usage is currently my preferred way
* Docker already running for my HA
* Create Container (may vary on the personal needs)
  * Dockerfile:

```
FROM node:latest
ENV NODE_ENV=development
EXPOSE 8000
WORKDIR /usr/src/app
COPY ["package.json", "package-lock.json*", "./"]
RUN npm install --development
RUN apt-get update
RUN apt-get install -y vim
COPY . .
CMD ["node", "server.js"]
```

* Build:
      
```
docker build --tag ecoflow-nodejs .
```

* Start Container:
    
```
docker run -d -p 8888:8000 --restart unless-stopped --name=ecoflow-nodejs ecoflow-nodejs
```
(Port 8888 for me because 8000 is already used elsewhere on my used Pi)

* Test by calling URL:
    
```
http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt=150&power_supply_mode=0
http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&power_supply_mode=0
http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt=150
```
(for 150W AC Output and Power Supply Mode to AC Output in one call or also individual call for only one option is possible)

* (Optional) Check the output from inside the container:

Docker Logs: 

```
docker logs --follow <containerID>

30/08/2023 08:09:10:  set Ac => 150 Watts
30/08/2023 08:09:10:  setPrio => 0
30/08/2023 10:47:08:  set Ac => 75 Watts
30/08/2023 10:47:09:  setPrio => 0
30/08/2023 10:47:44:  set Ac => 150 Watts
30/08/2023 10:47:44:  setPrio => 0
```

(docker logs timestamp is because of timezone/setting 2h behind)

### **Home Assistant**
* Home Assistant with an Webservice call to the URL of the Nodejs-Powerstream-server, therefore I created some (similiar to example on Bogdans readme):
    * input_numer (for AC Output Watts), input_text (for Mode) by HA Helpers functionalities
    * Webservice (powerstream_command) in configuration (in my case rest_commands.yaml)
    * Automation for state change of the input_numer (AC OutPut Watts) and input_text (Power Supply Mode) and calling Webservice (powerstream_command) 
    * For testing some buttons to change the input_numer (AC OutPut Watts) or input_text (Power Supply Mode)  
    * TODO: some intelligence/automation to change values / call Webservice in regards to my Smartmeter Live value in HA

* input_number/-text and some buttons created by Helpers in HA:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/374f623c-aef5-4e14-862f-126fd5c7349a)      
    * e.g. powerstream_ac_output:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/02e66cad-b02c-44f3-aa25-fffe08e3e1de) ![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/d959bf24-d156-4869-ba43-49a8feb0caa9)
 

 

* Webservices (powerstream_command...):
rest_command.yaml (included in configuration.yaml):
e.g.
```
powerstream_command: 
    url: "http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt={{ states('input_number.powerstream_ac_output') }}&power_supply_mode={{ states('input_text.powerstream_power_supply_mode') }}"
    verify_ssl: false

powerstream_command_set_ac_output: 
    url: "http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt={{ states('input_number.powerstream_ac_output') }}"
    verify_ssl: false

powerstream_command_set_prio_mode: 
    url: "http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&power_supply_mode={{ states('input_text.powerstream_power_supply_mode') }}"
    verify_ssl: false
```

* Automations:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/8e9d2ec1-393c-4016-8c14-feb5cffde902)
                
    * e.g. AC Output Watts
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/2c09ced1-4b75-4243-ba2d-dd49ef854e7f)
                
                              
* For Testing added the input_number/-text and buttons on HA card/view:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/ffdac92c-dc5b-4f97-b7b9-a28f9f247ec1) By pressing the buttons I can now change the AC Output Watts to 0, 75, 150 or the Power Supply Mode to Storage or AC Output/Grid. Also because the AC Output Watt (input_number) is defined as slider I can select the entity and change the output in between 0-600W (defined for input_number).

* First tests are working and I will go on testing and when it is working fine I will try to set AC Output Watts (Powerstream) depending on my live Smartmeter Current Power (Grid) value (or to what else is helping me in my home and local environment):
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/ad6b813c-3210-4755-89ac-4c528aff4685)


### **Work in Progress: HA Automations for trying to narrow zero to grid by controlling the Powerstream via the ecoflow-powerstream-nodejs from HA**
* created some Automations, WebServices, ... for controlling the Powerstream 
    * One Automation for trying to narrow the supply to grid near zero (currently called every 10 seconds)
    * working for first tests, optimization for some tactful devices like heating phase of oven maybe needed, ...
    * several other Automations for e.g. switching Zero Automation on/off, switching to Storage Mode if battery is full, switching to supply mode for the night to deliver AC output and empty the battery, ...
    * in nodejs (in my case the docker logs) monitor the calls to the ecoflow mqt API for analysing reasons
 
Now need to monitor if it is working fine and Automations are triggered, optimization afterwards (e.g. skip tactful devices) and thought about how to deal with battery in wintertime ;-)
  
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/d79c0a3d-cf03-42fd-8998-9ba51c6df1bd)
(docker logs timestamp is because of timezone/setting 2h behind)
 
### **Automation triggered when Delta 2 Max gets full** 
(currently 100% defined in Automation), inactivated zero automation (until next morning), switched to Storage mode to supply all solar to AC Output and in preparation set AC output to 150W for the night (Automation will be triggered at 20:00):
(potential overload solar for switching other devices on or maybe additional little battery, ... - but this overload will end at our location in a few days ...)
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/1b7f2109-241b-42e6-a606-d6e491d9b98b)
(docker logs timestamp is because of timezone/setting 2h behind)

### **Todays (07092023 1300) current state, Powerstream - Zero Automation was activated by Automation at 9:00 and supporting requests from ... ** 
Dishwascher, Washing Maschine, MicroWave with the maximum of 600W, also supporting different lower devices ...:
![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/bf795bcc-88d0-4173-9df8-03807c39271a)

### **Todays (07092023 1700) current state, Powerstream - Zero Automation was activated by Automation at 9:00 and deactivated at 16:30 (instead of Automation for full Delta2Max like yesterday)  ... **
deactivated at 16:30 because of solar input reduces now for the Powerstream panels and to prevent Delta2Max from unload before nightly supply, now in storage mode until 20:00 (will be optimized maybe with additional panels attached directly to D2M too...) 
![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/709c7511-cf57-43fe-a4d8-799093e92e02)


Comparison from yesterday (oven was running with requests to 600W) and today (2 x washing maschine, dishwasher were running and supported by 600W), today more devices with requests to more than 600W were running, weather was identical full sun (no clouds)

Yesterday: more supply to GRID (selfconsumed 89%) ![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/68c53149-a00d-4046-89cb-2cb3be7b93b0) ![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/c69abb8c-013d-49ca-9f8e-7278616eaca0)



Today: supply to GRID was minizized (selfconsumed >98%, at 17:00) ![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/9515996c-16b8-4918-aa2f-2dfbd3cfa2a7)  ![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/90f85092-4270-4c25-9bc0-6a7fbd6c66a4)



### **10092023 0900 current state **
Working very well ;-)

**But, depends of personell environment** and therefore ajustments to us the current available solar (e.g. I can switch additional consumers on or off directly in Home Assistant by Automations depending on the solar overload (pump water basin, water beds, air conditioner, ...; oven, microwave, dishwasher, coffee maker with higher effort are supported by 600W; )
Also we have a relatively high power consumption and can control were we want to consume it per HA Automations.


I have activated the HA control to try to narrow zero to GRID since three days now and the supply to GRID is minimist.
It was real good to compare here, because we had the whole last week best weather without any cloud ;-)
![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/03837767-0442-4497-b32a-39782217cab1)


View from Smartphone to Home Assistant and current settings/status:

![image](https://github.com/giovanne123/Ecoflow_Powerstream_HomeAssistant_Automation_Documentation/assets/16689453/23c1584d-667f-4e1d-9faa-27bfef4c435d)


But from now I will start thinking for the winter time and how to deal with it, for my environment it is now starting with bad solar production :-(
Also will check for optimizations... and follow what different solutions will maybe come up ... ;-)
