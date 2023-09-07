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
-	Cloned the repo from here (https://github.com/bogdancs92/ecoflow-powerstream-nodejs.git)
-	Nodejs Powerstream Config change: (values for e.g. URL, QUERY, TOKEN are free to use but must correspond to the REST-URL which will be called later) 
o	Create/edit “.env”
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
-	(optional) Nodejs Powerstream Container with nodejs-server waiting for request to change AC output Watts or Power Supply Mode on Powerstream
o	_Without container: “node server.js”_ can be used also without a container when nodejs is installed in OS, container usage is currently my preferred way
o	Docker already running for my HA
o	Create Container (may vary on the personal needs)
        Dockerfile:

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
        Build:
      

     `docker build --tag ecoflow-nodejs .`

o	Start Container
`docker run -d -p 8888:8000 --restart unless-stopped --name=ecoflow-nodejs ecoflow-nodejs`
(Port 8888 for me because 8000 is already used elsewhere on my used Pi)
o	Test by calling URL:
`http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt=150&power_supply_mode=0)`
(for 150W AC Output and Power Supply Mode to AC Output)
o	(Optional) Check the output from inside the container:
`docker logs --follow <containerID>`
```
30/08/2023 08:09:10:  set Ac => 150 Watts
30/08/2023 08:09:10:  setPrio => 0
30/08/2023 10:47:08:  set Ac => 75 Watts
30/08/2023 10:47:09:  setPrio => 0
30/08/2023 10:47:44:  set Ac => 150 Watts
30/08/2023 10:47:44:  setPrio => 0
```
(docker logs timestamp is because of timezone/setting 2h behind)

### **Home Assistant**
-	Home Assistant with an Webservice call to the URL of the Nodejs-Powerstream-server, therefore I created some (similiar to example on Bogdans readme):
o	input_numer (for AC Output Watts), input_text (for Mode) by HA Helpers functionalities
o	Webservice (powerstream_command) in configuration (in my case rest_commands.yaml)
o	Automation for state change of the input_numer (AC OutPut Watts) and input_text (Power Supply Mode) and calling Webservice (powerstream_command) 
o	For testing some buttons to change the input_numer (AC OutPut Watts) or input_text (Power Supply Mode)  
o	TODO: some intelligence/automation to change values / call Webservice in regards to my Smartmeter Live value in HA

- input_number/-text and some buttons created by Helpers in HA:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/374f623c-aef5-4e14-862f-126fd5c7349a)
                    
e.g. powerstream_ac_output:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/02e66cad-b02c-44f3-aa25-fffe08e3e1de)

![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/d959bf24-d156-4869-ba43-49a8feb0caa9)
 

 

- Webservice (powerstream_command):
Rest_command.yaml (included in configuration.yaml):
```
powerstream_command: 
    url: "http://192.168.0.xxx:8888/cmd?my_token=my_secret_for_token&ac_output_watt={{ states('input_number.powerstream_ac_output') }}&power_supply_mode={{ states('input_text.powerstream_power_supply_mode') }}"
    verify_ssl: false
```

- Automations:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/8e9d2ec1-393c-4016-8c14-feb5cffde902)
                
e.g. AC Output Watts
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/2c09ced1-4b75-4243-ba2d-dd49ef854e7f)
                
                              
- For Testing added the input_number/-text and buttons on HA card/view:
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/ffdac92c-dc5b-4f97-b7b9-a28f9f247ec1)
 

By pressing the buttons I can now change the AC Output Watts to 0, 75, 150 or the Power Supply Mode to Storage or AC Output/Grid.
Also because the AC Output Watt (input_number) is defined as slider I can select the entity and change the output in between 0-600W (defined for input_number).

First tests are working and I will go on testing 
and when it is working fine I will try to set AC Output Watts (Powerstream) depending on my live Smartmeter Current Power (Grid) value (or to what else is helping me in my home and local environment):

![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/ad6b813c-3210-4755-89ac-4c528aff4685)

**Work in Progress: HA Automations for trying to narrow zero to grid by controlling the Powerstream via the ecoflow-powerstream-nodejs from HA**
- created some Automations, WebServices, ... for controlling the Powerstream 
  - One Automation for trying to narrow the supply to grid near zero (currently called every 10 seconds)
  - working for first tests, optimization for some tactful devices like heating phase of oven maybe needed, ...
  - several other Automations for e.g. switching Zero Automation on/off, switching to Storage Mode if battery is full, switching to supply mode for the night to deliver AC output and empty the battery, ...
  - in nodejs (in my case the docker logs) monitor the calls to the ecoflow mqt API for analysing reasons
 
Now need to monitor if it is working fine and Automations are triggered, optimization afterwards (e.g. skip tactful devices) and thought about how to deal with battery in wintertime ;-)
  
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/d79c0a3d-cf03-42fd-8998-9ba51c6df1bd)
(docker logs timestamp is because of timezone/setting 2h behind)
 
**Automation triggered when Delta 2 Max gets full** (currently 100% defined in Automation), inactivated zero automation (until next morning), switched to Storage mode to supply all solar to AC Output and in preparation set AC output to 150W for the night (Automation will be triggered at 20:00):
(potential overload solar for switching other devices on or maybe additional little battery, ... - but this overload will end at our location in a few days ...)
![image](https://github.com/bogdancs92/ecoflow-powerstream-nodejs/assets/16689453/1b7f2109-241b-42e6-a606-d6e491d9b98b)
(docker logs timestamp is because of timezone/setting 2h behind)
