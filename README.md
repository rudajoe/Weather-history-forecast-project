# Weather-history-forecast-project
part 1: asking for the user's request
part 2: getting the data
part 3: displaying the data
part 4: displaying the data graphically (charts + wordcloud)
part 5: saving figures in html and pdf formats

-> output of this project will be a "data" directory with: 
   5 .png figures, 1 .pdf file, 1 .html file, 1 .csv file and 1 .json file

"""

#%%

import datetime
import os 
import requests
import json
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.backends.backend_pdf import PdfPages
from wordcloud import WordCloud
from spacy.lang.fr.stop_words import STOP_WORDS as fr_stop


def validate(date):

    try:
        datetime.datetime.strptime(date, '%Y-%m')
        return True
    except:
        #print("The format is incorrect, it should be YYYY-MM. Please enter date in the right format: \n")
        return False


#%%

### user request ###

# get current date 

currentDate = datetime.date.today()
print(currentDate) 
Date = currentDate.strftime('%d %b %Y ')
print(Date)


# display welcome message

print("Hello, today's date is,", Date, "\n", "To retrieve the measurements you need,", "\n", "we will ask you for a weather station name and date.")


# ask the weather station name + make it upper case

print("\n Firstly, please enter the weather station name below: \n")
station = input()
station = str(station) 

 
Station = station.upper() 
print("You have selected the weather station:", Station)


# ask for user's month date

print("\n Please enter the month date for which you would like to retrieve the data: \n (the year format should be the following: YYYY-MM)")
date = input()
 

while validate(date) == False:
    print("The date is not in the correct format. It should be the following: YYYY-MM)")
    date = input()

print("The date", date, "you have selected is valid.")

#%%

### getting the data ###

# download the dataset selected by the user in Json format

url = "https://data.opendatasoft.com/api/records/1.0/search/?dataset=donnees-synop-essentielles-omm%40public&q=&rows=10000&sort=-date&facet=date&facet=nom&facet=temps_present&facet=libgeo&facet=nom_epci&facet=nom_dept&facet=nom_reg&refine.nom="+ Station+ "&refine.date=" + date


r = requests.get(url, verify=False)
jdata = r.json()

# display the correctly formatted dataset

weatherData = json.dumps(jdata, indent= 3)
print(weatherData)

# create a folder called 'data'

directory = "data"

current_dir = os.getcwd()

path = os.path.join(current_dir, directory)

try :
    os.makedirs(path, exist_ok = True)
    print("Directory '%s' created successfully" %directory)
except OSError as error:
    print("Directory '%s' cannot be created" %directory)
    
# save data as a JSon file in the 'data' directory 

os.chdir("data")

with open('weatherdata.json', 'w') as json_file:
    json.dump(jdata, json_file)

#%%

### displaying the data ###

# display the data downloaded

with open('weatherdata.json', 'r') as json_file:
	j_load = json.load(json_file) 
    

stationNumber = j_load["records"][0]["fields"]["numer_sta"]
stationNumber = int(stationNumber)

print("Weather Station:", Station)
print("Station Number:",stationNumber)
print("Today's date:", Date)
print("Chosen month:", date)
print("* Tmin(??C) is the lowest temperature recorded in the last 12 hours.")
print("* Tmax(??C) is the highest temperature recorded in the last 12 hours.")
print("* u refers to humidity.")
print("|----------------------------------------------------|")
print("|   Date   | Time | T(??C) | u(%) | Tmin(??C)| Tmax(??C)|")
print("|----------------------------------------------------|")

for a in j_load["records"]:
    try:
        tmin = a["fields"]["tn12c"]
        tmin = round(tmin, 2)
        tmax = a["fields"]["tx12c"]
        tmax = round (tmax, 2)
    except:
        tmin = "NA  "
        tmax = "NA  "
    finally:
        m = a["fields"]["date"][0:10]
        h = a["fields"]["date"][11:16]
        t = a["fields"]["tc"]
        t = round(t, 2)
        u = a["fields"]["u"]
        print("|", m, h, "|", t, "|", u,"  |", tmin,"   |", tmax,"   |")

print("|----------------------------------------------------|")

# putthe json file data in a python dataframe

j_load

df = pd.json_normalize(j_load,record_path=['records'])

# create a csv file + save it in the 'data' folder

df.to_csv("weatherdata.csv") #file used for tableau figure

#%% 

### Display charts with Python matplotlib or seaborn ###

 
df.rename(columns={'fields.date': 'date'}, inplace=True)
df.rename(columns={'fields.tc': 'temp'}, inplace=True)
df.rename(columns={'fields.u': 'hum'}, inplace=True)
df.rename(columns={'fields.tn12c': 'Tmin'}, inplace=True)
df.rename(columns={'fields.tx12c': 'Tmax'}, inplace=True)
df.rename(columns={'fields.temps_present' : 'description'}, inplace=True)

df[['date','time']] = df.date.str.split("T",expand=True,)
df['date']= pd.to_datetime(df['date'])


def tmin(row):
   if row['time'] == "06:00:00+00:00":
      return row['Tmin']

def tmax(row):
    if row['time'] == "18:00:00+00:00":
        return row['Tmax']

def temp(row):
    if row['time'] == "12:00:00+00:00":
        return row['temp']

def hum(row):
    if row['time'] == "12:00:00+00:00":
        return row['hum']
    
df['Tmin'] = df.apply(lambda row: tmin(row), axis=1)
df['Tmax'] = df.apply(lambda row: tmax(row), axis=1)
df["Temp"] = df.apply(lambda row: temp(row), axis=1)
df["Hum"] = df.apply(lambda row: hum(row), axis= 1)


# chart 1

x = "temp"
y = "hum"

df.plot(x, y, kind = "scatter", figsize =(20,10))
plt.xlabel("Temperature (??C)")
plt.ylabel("Humidity (%)")
plt.title("Temperature and Humidity in "+Station+" in "+date, fontsize='20')
plt.savefig("chart1.png")

# chart 2
 
x = df.date
y = df.Temp

plt.figure(figsize=(20,10))
plt.stem(x, y, use_line_collection = True)
markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
plt.xticks(x)
plt.xticks(rotation=90)
plt.ylabel("Temperature (??C)")
plt.title("Temperature every 24 hours in "+Station+" during "+date, fontsize='20')
plt.savefig("chart2.png")

# chart 3

x = df.date
y = df.Hum
plt.figure(figsize=(20,10))
plt.stem(x, y, use_line_collection = True)
markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
plt.xticks(x)
plt.xticks(rotation=90)
plt.xlabel("Date")
plt.title("Humidity (%) in "+Station+" during "+date, fontsize='20')
plt.savefig("chart3.png")

# chart 4

x = df.date
y = df.Tmin
plt.figure(figsize=(20,10))
plt.stem(x, y, use_line_collection = True)
markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
plt.xticks(x)
plt.xticks(rotation=90)
plt.ylabel("Temperature (??C)")
plt.title("Minimum temperature every 24hrs in "+Station+" during "+date, fontsize='20')
plt.savefig("chart4.png")

# chart 5

x = df.date
y = df.Tmax
plt.figure(figsize=(20,10))
plt.stem(x, y, use_line_collection = True)
markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
plt.xticks(x)
plt.xticks(rotation=90)
plt.ylabel("Temperature (??C)")
plt.title("Maximum temperature every 24hrs in "+Station+" during "+date, fontsize='20')
plt.savefig("chart5.png")


#%% 

### Add data, text, and charts in a generated multi-page PDF ###

pdf = PdfPages("charts.pdf")

try: 
    firstPage = plt.figure(figsize=(11.69,8.27))
    txt = ("Collection of figures for data collected in "+date+" in "+Station)
    firstPage.text(0.5,0.5,txt, transform=firstPage.transFigure, size=24, ha="center")
    pdf.savefig()  
    fig = plt.figure(figsize=(11.69,8.27))
    txt1 = "Figure 1: relationship between temperature and humidity \n\n" \
        "Figure 2: temperature change over month (1 measurement/day) \n\n"\
            "Figure 3: humidity change over month (1 measurement/day) \n\n"\
                "Figure 4: minimum temperature every 24hrs over month \n\n"\
                    "Figure 5: maximum temperature every 24hrs over month"
    fig.text(0.5, 0.7, txt1, transform=firstPage.transFigure, size=22, ha="center",  va="top")
    pdf.savefig()
    x = "temp"
    y = "hum"
    df.plot(x, y, kind = "scatter", figsize =(20,10))
    plt.xlabel("Temperature (??C)")
    plt.ylabel("Humidity (%)")
    plt.title("Temperature and Humidity in "+Station+" in "+date, fontsize='20')
    pdf.savefig()
    x = df.date
    y = df.Temp
    plt.figure(figsize=(20,10))
    plt.stem(x, y, use_line_collection = True)
    markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
    plt.xticks(x)
    plt.xticks(rotation=90)
    plt.ylabel("Temperature (??C)")
    plt.title("Temperature every 24 hours in "+Station+" during "+date, fontsize='20')
    pdf.savefig()
    x = df.date
    y = df.Hum
    plt.figure(figsize=(20,10))
    plt.stem(x, y, use_line_collection = True)
    markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
    plt.xticks(x)
    plt.xticks(rotation=90)
    plt.xlabel("Date")
    plt.title("Humidity (%) in "+Station+" during "+date, fontsize='20')
    pdf.savefig()
    x = df.date
    y = df.Tmin
    plt.figure(figsize=(20,10))
    plt.stem(x, y, use_line_collection = True)
    markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
    plt.xticks(x)
    plt.xticks(rotation=90)
    plt.ylabel("Temperature (??C)")
    plt.title("Minimum temperature every 24hrs in "+Station+" during "+date, fontsize='20')
    pdf.savefig()
    x = df.date
    y = df.Tmax
    plt.figure(figsize=(20,10))
    plt.stem(x, y, use_line_collection = True)
    markerline, stemlines, baseline = plt.stem(x, y, linefmt ='gainsboro')
    plt.xticks(x)
    plt.xticks(rotation=90)
    plt.ylabel("Temperature (??C)")
    plt.title("Maximum temperature every 24hrs in "+Station+" during "+date, fontsize='20')
    pdf.savefig()
    pdf.close()
except:
    pdf.close()
    

#%%

### Generate an .html ###

f = open("summary.html", "w")

html = ("<html>\n<head>\n<title>\nOutput in HTML file\</title>\n</head><body><hl><table><img>")
html += "<tbody><center><bold>"+"Summary of weather data concerning your request"+"</bold></body>"

for record in j_load["records"]:

    html += "<tr>"
    a = record["fields"]["date"][0:10]
    html += "<td>"+str(a)+"</td>"
    b = record["fields"]["tc"]
    b = round(float(b),2)
    html += "<td>"+str(b)+"</td>"
    c = record["fields"]["u"]
    html += "<td>"+str(c)+"</td>"
    e = record["fields"]["pres"]
    html += "<td>"+str(e)+"</td>"
    g = record["fields"]["temps_present"]
    html += "<td>"+str(g)+"</td>"   
    html += "</tr>"
    
html += "<img>"
html += "<img src=chart1.png>"
html += "<img src=chart2.png>"
html += "<img src=chart3.png>"
html += "<img src=chart4.png>"
html += "<img src=chart5.png>"
html +="</img>"

html += "<tbody>"+"|   Date      | Temperature | Humidity | Description of weather |"+"</body>" 

f.writelines(html)
f.close()

##%%%

# Displaying a wordcloud

text = " ".join(review for review in df.description.astype(str)) #text list with all words in column "description"
#print ("There are {} words in the combination of all cells in column description.".format(len(text)))
stopwords = list(fr_stop)
wordcloud = WordCloud(stopwords=stopwords, background_color="white", width=800, height=400).generate(text)
plt.figure(figsize=(20,10))
plt.axis("off")
plt.tight_layout(pad=0)
plt.imshow(wordcloud, interpolation='bilinear')
plt.show()
