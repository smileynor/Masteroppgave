# A scheme for real-time monitoring of power oscillation in a power grid
This is the **README** for my project code used during my final master at NTNU.

## Table of Content:
The monitoring system consist of multiple files, each performing it's own task.

* CollectingData.py
TO BE WRITTEN!
* GeneratePSSEcode.py
TO BE WRITTEN!
* RunPSSEcode.py
TO BE WRITTEN!
* Post-Visualization
TO BE WRITTEN!

## Dependencies
TO BE WRITTEN!
A good start is to install [Anaconda 2.7], [PSSE], [PacDyn].  

## Installation
To run the monitoring system, multiple dependences is needed. The _script_ is written in `Python 2.7`, and used the simulation tools `PSS/E 33` for load flow studies, and `PacDyn 9.8.0 or later` for small-signal studies. The two latter is commersial programs, and a licence need to be obtained. **Note:** Running the `CollectingData` file also requires access to NordPoolSpot`s server. I'm not eligble to distribute the password, so this has to be _aquired from them personally_.

## Collecting Data
The script `CollectingData.py` connects to _NordPoolSpot_'s ftp-server to collect historical reports from the Nordic power grid. These are _weekly reports_ with historical information about production, consumption, flow between areas, prices etc. Each country has their own report, so to work on a system consisting of more than one country a merge had to be performed. 
The returned files from _NordPoolSpot_ has a list of entries, each line consist of 32 fields, following the syntax:

```
Datatype;Code;Year;Weeknumber;DayOfWeekStartingAtOneOnMondays;DateWithFormat(DD.MM.YYYY);AreaCode;Hour1;Hour2;Hour3A;Hour3B;Hour4;Hour5;...;Hour23;Hour24;SumDay
```

**Note**: The fields have a `Hour3A` and `Hour3B` to remedy for **summer time**. During the _spring time correction_ both of the entries will be evaded, while at _fall time correction_ both fields are used. 

**Known bug**: The fact, stated in the previous note, was not recogniced from the beginning, and the program kept crashing when trying to collect the `Hour3A` at the end of March. To remedy the crash, a simple `try-catch` statement was inserted to handle all crashes that could happen during the `CollectingData`. The _script_ will also skip the `Hour3B` altogether, since it was thought to be unused. **This should be fixed when using the code, to get proper answers**.

#### File dependency
As stated in the header of the file, the code need to import some dependencies:
```python
import datetime
import numpy as np
import csv
from ftplib import FTP
from IPython.lib.security import passwd
import urllib
```

#### Ftp-credentials (Needs to be edited, fill with your own)
The FTP-login details could not be distribbuted with the codes, and therefore a placeholder was created. Locate the following code:
```python
FTPstudentlogin = 'student'
FTPstudentpassword = '********'
collectFromFTPserver = True
```
and change `'******'` into password provided from _NordPoolSpot_ as `'Password'`. To be able to run the _script_ offline, the ftp-server was cloned temporary, giving the choice of collecting the data locally (by changing `collectFromFTPserver` to `False`) or externally at the FTP-server.

#### Functions

The file has multiple functions, created for different use. 

##### CalculateCollectingData(caseDateAndHour)

Used when the spesified time could be between a full hour, returning an interpolated result of the flow, production and consumption. Both return _numpy_-arrays and used `saveData()` to create csv
```python
12 def calculateCollectingData(caseDateAndHour,saveProductionToPath=0,saveHVDCtoPath=0,saveFolderPath=0):
```
* Input: (caseDateAndHour) _datetime.datetime_ formatted value of the date and time which data should be obtained for.
* Output: (HVDC_flow,Production_And_Consumption)  _numpy_-array formatted value of the flow, and production/consumption.
>**Example:** 3.march 2013 11:20 
>would collect data for 3.march 2013 at 11:00 and at 12:00, returning a interpolated value of 1/3 of the change from 11:00 to 12:00. 

**Note**: From earlier versions of the function, `saveProductionToPath=0`, `saveHVDCtoPath=0`, `saveFolderPath=0`, were used to tell the program where to save the results in .csv files. These have _no function_ anymore, and **could be deleted**. 

##### interpolateCollectedData(H0,P0,H1,P1,caseDateAndHour):
Used to perform the interpolation of two cases. 
```python
28 def interpolateCollectedData(H0,P0,H1,P1,caseDateAndHour):
```
* Input: (first_HVDC_flow, first_ProductionAndConsumption, second_HVDC_flow, second_ProductionAndConsumption, datetime)
, 4 _numpy_-array formatted results obtained from `collectData()`, and a _datetime.datetime_ formatted time.
* Output: (HVDC_flow,ProductionAndConsumption) _numpy_-array formatted results.

##### saveData(H,PC, saveHVDCtoPath,SaveP&CtoPath)
Writes two `.csv` files containing the HVDC flow and the areas production/consumption into respective files.
```python
53 def saveData(listOfHVDC, outputToCurrentProductionAndLoadForScaling, saveHVDCtoPath='currentHVDCData.csv', saveProductionToPath='currentProductionAndLoadForScaling.csv'):
```
* Input: (HVDC, Production_Consumption, saveHVDCtoPath, SaveProd&ConsumptionToPath)
* `HVDC` and `Production_Consumption` is _numpy_-arrays
* `saveHVDCtoPath` and `SaveProd&ConsumptionToPath` is _optional_ _strings_, showing absolute or relative path to where to save the `.csv` files. If no input is provided, it will be saved to the same folder as the code is runned from into the files: `'currentHVDCData.csv'` and `currentProductionAndLoadForScaling.csv`

The **HVDC .csv file** will contain a table with a header and a line for each HVDC point following the format:

Bus | ID | Flow
--- | --- | ---
5000 | 1 | 450
5500 | 1 | -300
... | ... | ...
Here the `Bus` is the `PSS/E` bus number, `ID` is the load ID for the specified bus, and a positive `Flow` tells how much is being imported through this HVDC point. 

The **Production and Consumption .csv file** will contain a table with a header and a line for each area following the format:
Area | Production | Consumption
--- | ---: | ---:
11 | 12345 | 9876
12 | 8765 | 11223
... | ... | ...

Here the `Area` correspond to the area number in `PSS/E`, and `Production` and `Consumption` is the total area production and consumption of the corresponding area.
##### collectData(caseDateAndHour):
This is where the magic is happening. This is the function that will perform the ftp-call, collect, merge and filter the reports into _numpy_-arrays. 
**Note**: To be able to call this function, the ftp-password or credentials need to be updated in the start of the code file. See section (**Ftp-credentials**)

```python
70 def collectData(caseDateAndHour):
```

* Input: (caseDateAndHour) _datetime.datetime_ formatted value of the date and time which data should be obtained for.
* Output: (HVDC_flow,Production_And_Consumption)  _numpy_-array formatted value of the flow, and production/consumption.

If your goal is to only use the function to collect data for the nordic grid, consisting of _Norway_, _Sweden_ and _Finland_, plus the HVDC connections to this area: the following code could be used

**Minimal working example,**
run a python file with the following code:
```python
 import datetime
 import collectingData
 
caseDateAndHour = datetime.datetime.strptime('11.03.2015-08:30','%d.%m.%Y-%H:%M')
HVDCdata,ProdConData = collectingData.collectData(caseDateAndHour)
print(HVDCdata)
print(ProdConData)
```

>The code will create a _datetime.datetime_ formatted time equal to 11. March 2015 and collect the data for the 8'th hour. Using this `collectData(caseDateAndHour)` will only give the result for the 8'th hour, and will not interpolate to obtain a moment production. The same result would be obtained if `11.03.2015-08:00` was sent. To get a interpolated result, use the function `collectingData.calculateCollectingData(caseDateAndHour)` instead.

###### #The code explained

**The paths on the ftp-server**, where the reports should be collected from is created. The paths are obtained by combining country, year, week. The year and week are drawn from the _datetime.datetime_ variable provided. Older years are saved to their own folders. 
```python
79     # Alternative paths if before 2016
80    pathNorway = '/Operating_data/Norway/'
81    pathSweden = '/Operating_data/Sweden/'
82    pathFinland = '/Operating_data/Finland/'
83    yearNumber = datetime.datetime.strftime(caseDateAndHour,'%y')
84    weekNumber = "%02d" % caseDateAndHour.isocalendar()[1]
85    pathEnding = yearNumber + weekNumber + '.sdv'

87    if caseDateAndHour.year < 1998:
88        print('No data before 1997 exist, program is quitting')
89        exit()
90    elif caseDateAndHour.year < 2016:
91        pathNorway = pathNorway + str(caseDateAndHour.year) + '/pono' + pathEnding
92        pathSweden = pathSweden + str(caseDateAndHour.year) + '/pose' + pathEnding
93        pathFinland = pathFinland + str(caseDateAndHour.year) + '/pofi' + pathEnding
94    elif caseDateAndHour.year == 2016:     # This should be changed to update to todays year
95        pathNorway = pathNorway + '/pono' + pathEnding
96        pathSweden = pathSweden + '/pose' + pathEnding
97        pathFinland = pathFinland + '/pofi' + pathEnding
```
A placeholder for all the _weekly reports_ are temporary created.
```python
105    lines = []
```
**Collecting all the relevant reports** from _NordPoolSpot_ and appending it to `lines`. **Note**: the `FTPstudentpassword` should have been specified
```python
106    def collectFromFTP():
107        # Connecting to the FTP-server
108        ftp = FTP('ftp.nordpoolspot.com')
109        ftp.login(user=FTPstudentlogin, passwd =FTPstudentpassword)
110        ftp.retrlines('RETR '+ pathNorway, lines.append)
111        ftp.retrlines('RETR '+ pathSweden, lines.append)
112        ftp.retrlines('RETR '+ pathFinland, lines.append)
...        ...
125    if collectFromFTPserver == True:
126        collectFromFTP()
```
The `lines` are then manipulated into looking like a `.csv` file to be able to use `csv.reader` on it. 
```python
134    csvfile = ''
135    for line in lines:
136        csvfile = csvfile + line + '\n'

137    # For deleting the last '\n' to prevent errors while reading an empty line.
138    csvfile = csvfile[:-2]
```

All the data that is needed to give a snapshot of the situation during the spesified time is now inside the variable `csvfile`. A class is created to be able to hold just the relevant information about the specified time, dropping all unrelevant data. In our example this object-variable is called `testCase`. 

```python
72    class StudyCase(object):
73        def __init__(self,dateAndHour):
74            self.P = {}
75            self.C = {}
76            self.FLOW = {}
77            self.caseTime = dateAndHour
:               :
141    testCase = StudyCase(caseDateAndHour)
:               :
145    # Initializing all flow keys
146    testCase.FLOW['FI_EE'] = 0
147    testCase.FLOW['FI_RU'] = 0
148    testCase.FLOW['FI_SE3'] = 0
149    testCase.FLOW['NO_DK'] = 0
150    testCase.FLOW['NO_NL'] = 0
151    testCase.FLOW['SE3_DK1'] = 0
152    testCase.FLOW['SE3_FI'] = 0
153    testCase.FLOW['SE4_DK2'] = 0
154    testCase.FLOW['SE4_DE'] = 0
155    testCase.FLOW['SE4_PL'] = 0
```

The `.FLOW[area]` needs to be initialized because later multiple HVDC points are added to the same _Bus number_ (bus node). If the program tried to add a specified HVDC cable that had yet not been built, the program could not find the HVDC-point (for good reasons), and the program would crash, trying to look up a _dictionary key_ that did not exist.

Up to this point, the reports consist of all available data for the specified week and now it is time to **filter out** the data to only contain data that is used further, in this case the _production_, the _consumption_ and the _HVDC flow_ in the Nordic grid for the specified time.

```python
157    # Because of summer time, a shift is needed to collect the time correctly. 
158    hourOfInterest = caseDateAndHour.hour
159    if caseDateAndHour.year >=2014:
160        if hourOfInterest > 2:
161            hourOfInterest +=1

163    dateOfInterest = datetime.datetime.strftime(caseDateAndHour,'%d.%m.%Y')

165    # For splitting the data into cells
166    reader = csv.reader(csvfile.split('\n'), delimiter=';')
167    for row in reader:
168        if row[0] =='PS':
169            if row[1] =='P':
170                if row[5] == dateOfInterest:
171                    #print(row)
172                    try:
173                        testCase.P[row[6]]=int(row[hourOfInterest+7])
174                    except:
175                        pass
176        if row[0] =='FB':
177            if row[1] =='F':
178                if row[5] == dateOfInterest:
179                    #print(row)
180                    try:
181                        testCase.C[row[6]]=int(row[hourOfInterest+7])
182                    except:
183                        pass
184        if row[0] =='UT':
185            if row[1] =='U':
186                if row[5] == dateOfInterest:
187                    #print(row)
188                    try:
189                        testCase.FLOW[row[6]]=int(row[hourOfInterest+7])
190                    except:
191                        pass
192    # Using "With open" deletes a variable after use, here the function "del" is used instead.
193    del reader
```
These nested if-sentences will look through each line of `csvfile` (earlier `lines`). Data is copied to the _class object_ `testCase` if relevant information is found. Following the _code syntax_ of the files at the FTP:

```
Datatype;Code;Year;Weeknumber;DayOfWeekStartingAtOneOnMondays;DateWithFormat(DD.MM.YYYY);AreaCode;Hour1;Hour2;Hour3A;Hour3B;Hour4;Hour5;...;Hour23;Hour24;SumDay
```
If the _first field_ of a line contain the _datatype_ for production (`PS`), consumption (`FB`) or flow (`UT`) , the _second field_ is cheched if it contains actual data and not just _predicted data_, by looking for (`P`),(`F`),(`U`). Next, the _5th field_ is checked to see if it correspond to the correct date `dateOfInterest` (the same date as the input to the function).


_First field_ (`Datatype`) options:
* `PR`, (Prices in EUR/MWh)
* `OM`, (Turnover in MWh/h)
* `FB`, (Consumption in MWh/h)
* `PS`, (Total production in MWh/h)
* `UT`, (Net exchange in MWh/h)

_Second field_ (`Code`) options:
* `RN`, (Down regulating)
* `RO`, (Regulated market Up)
* `RC`, (Imbalance price consumption, used in settlement)
* `RP`, (Imbalance price production, purchase)
* `RS`, (Imbalance price production, sale)
* `DD`, (Dominated direction, (1) = up regulation, 0 = no regulation, (-1) = down regulation
* `F`, (Total consumption)
* `E`, (Estimated consumption for the following day)
* `P`, (Total production)
* `PE`, (Estimated production for the following day)
* `U`, (Net exchange, import = (+), export = (-))

For more details about the file format, see `Information/File_specifications` in _NordPoolSpot_'s server.


   [PSSE]: <http://w3.siemens.com/smartgrid/global/en/products-systems-solutions/software-solutions/planning-data-management-software/planning-simulation/pages/pss-e.aspx>
   [PacDyn]: <http://www.cepel.br/produtos/pacdyn-analise-e-controle-de-oscilacoes-eletromecanicas-em-sistemas-de-potencia.htm>
   [Anaconda 2.7]: <https://www.continuum.io/downloads>

## 

License
----

MIT


