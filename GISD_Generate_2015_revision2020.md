---
title: "GISD - German Index of Socio-Economic Deprivation"
author: "Niels Michalski"
date: "04 04 2020"
output: 
  html_document:
    keep_md: true
    toc: true
    toc_float: false
    toc_depth: 2
---



# Intro
...

## Fehlerbehebung

Fehlerbehebung in der Revision 2019  
von Niels Michalski

Hintergrund Fehlerbehebung:
In der Revision 2019 der GISD-Daten tauchten zum Teil starke Abweichungen zur Revision 2018 auf.Bei der Addition der Teildimensionen ging die Bildungsdimension in umgekehrter Richtung in den GISD ein. Mehrere Faktoren spielten dabei eine Rolle:

1. Die Struktur der Faktorladungen der Bildungsindikatoren ist nicht robust gegenüber Datenschwankungen. 
  * Der erste Faktor bildet die intendierte Kompomente ab. Es gibt allerdings einen zweiten Faktor mit Eigenwert über 1. 
  * Die Gewichte der Faktorladungen der Indikatoren BeschaeftigteohneAbschluss und Schulgaengerohneabschluss variieren sehr stark zwischen den Revisionen 2018 und 2019 
2. Die Indikatoren BeschaeftigteohneAbschluss und BeschaeftigtemitHochschulabschluss dieser Teildimension weisen die höchsten Anteile an MissingData auf (75%).
3. Ausschlaggebend ist am Ende eine negative Korrelation der Bildungsdimension mit dem Anteil Arbeitsloser, der zur Umkehrung des Faktors führt. Der Anteil Arbeitsloser wird als Markerindikator verwendet. 

Eine Möglichkeit: Der Indikator BeschäftigteohneAbschluss wird durch SchulabgängermitHochschulreife ersetzt.

## SOP for Revision (nach Lars Kroll)
1. Obtain new Data and Reference Files from INKAR (manually) -> check format
2. Change Year in GISD_generate_postcodes.R according to INKAR Regional Date and execute
3. Execute GISD_Generate.R (there should be no edits required)


# Beschreibung der Syntax
## 0. Benötigte Pakete


```r
library("tidyverse") # Tidyverse Methods
library("readxl") # Read Excel
library("imputeTS") # Impute Missing Features
library("haven") # write Stata-dta
library("sf") # write Stata-dta

# Create Output directories in working directory if necessary
dir.create("Outfiles")
dir.create("Outfiles/2019")
dir.create("Outfiles/2019/Bund")
dir.create("Outfiles/2019/Other")
dir.create("Outfiles/2019/Stata")
```

## I.  Generierung eines ID-Datensatzes

Generierung eines Datensatzes in dem den kleinsten regionalen Einheiten (Gemeinden) alle
übergeordneten regionalen Einheiten und deren Regionalkennziffern zugeordnet werden. 
Datenquelle ist die Gebietsstandsreferenz von Destatis.


```r
Gemeinden_INKAR <- read_excel("Data/Referenz/Referenz_1998_2015.xlsx", sheet = "Gemeinden", na = "NA", skip = 2) %>% 
  rename(Kennziffer=gem15,"Kennziffer Gemeindeverband"="Gemeindeverband, Stand 31.12.2015") %>% filter(!is.na(Kennziffer))
# Pipes: 
# 1. rename von zwei Variablen; " um Leerzeichen zu berÃ¼cksichtigen; 
# 2. Gemeinden ohne Missing auf der Kennziffervariablen

Gemeindeverbaende_INKAR <- read_excel("Data/Referenz/Referenz_1998_2015.xlsx", sheet = "GVB 2015", na = "NA", skip = 2) %>% 
  select("Kennziffer Gemeindeverband"=gvb15,"Name des Gemeindeverbands") %>% filter(!is.na("Kennziffer Gemeindeverband")) 
# das ganze nochmal für Gemeindeverbaende  
# Pipes: 
# 1. nur die Variablen gvb15 und Name des Gemeindeverbands ausgewÃ¤hlt 
# 2. Missing herausfiltern

Kreise_INKAR <- read_excel("Data/Referenz/Referenz_1998_2015.xlsx", sheet = "Kreise", skip = 2) %>%
 mutate(Kennziffer = as.numeric(krs15)/1000) %>% filter(!is.na(Kennziffer))
# und für Kreise
# Pipes: 
# 1. neue Variable generieren, die die Kreisvariable auf den FÃ¼nfsteller reduzieren
# 2. Missing herausfiltern


# Die drei Datensätze werden nun ausgehend vom Gemeindedatensatz zu einem ID-Datensatz zusammmengefügt
id_dataset <- Gemeinden_INKAR %>% 
              select(Gemeindekennziffer=Kennziffer,"Name der Gemeinde"=`Gemeindename 2015`,"Kennziffer Gemeindeverband") %>% 
              mutate(Kreiskennziffer=floor(Gemeindekennziffer/1000)) %>%
              left_join(.,Kreise_INKAR %>% select(Kreiskennziffer=krs15,
                                                  "Name des Kreises"=krs15name,
                                                  "Raumordnungsregion Nr"=ROR11,
                                                  Raumordnungsregion=ROR11name,
                                                  NUTS2,
                                                  "NUTS2 Name"=NUTS2name,
                                                  Bundesland=...24) %>% 
                          mutate(Kreiskennziffer=floor(Kreiskennziffer/1000)),by="Kreiskennziffer") %>%
              left_join(.,Gemeindeverbaende_INKAR, by="Kennziffer Gemeindeverband")

# Pipes:  1. (select) Variablenauswahl (gkz, Gemeindename, Gemeindeverband)[wieso hier ``?]
#         2. die Kreiskennziffer wird aus der Gemeindekennziffer generiert; floor rundet nach unten auf ganze Ziffern ab
#         3. leftjoin spielt Kreisdaten über Kreiskennziffer an
#         3.1 select wählt, die anzupielenden Variablen aus, darunter auch NUTS und ROR und Bundesland, dessen Variablenname beim       #             Einlesen zu lang war (...24)
#         3.2 die Kreiskennziffer wurde vor dem leftjoin im Using-Datensatz generiert
#         4. als letztes werden die Gemeindeverbandskennziffern angespielt
```


## II. Erzeugen eines Datensatzes mit Kennziffern als ID unabhängig von der Ebene 

Die Information für die Indikatoren, die für die Berechnung des GISD verwendet werden, liegt auf unterschiedlichen Ebenen vor. Die Faktorenanalyse soll auf Gemeindeebene durchgeführt werden. Percentile aus dem Index sollen am Ende für jede regionale Ebene separat berechnet werden können. 
Datenbasis sind die INKAR-Daten der jeweiligen Indikatoren im Excel-Format


```r
# Basis erzeugen: Ausgangspunkt Kreisdaten
# all levels of input will be added, Kreise is just a starting point.
Basedata    <- Kreise_INKAR %>% select(Kennziffer) %>% mutate(Jahr=2015)
# Datensatz zum Anspielen der Daten generieren
# Ausgangspunkt Kreisdatensatz
# Pipes:  1. nur Kreiskennzifern ausgewÃ¤hlt
#         2. Jahresvariable generiert (2015)

# Liste der Variablen generieren
inputdataset <- list.files("Data/INKAR_1998_2015") # Variablenliste der Dateinamen im Ordner



# Einlesen der einzelnen Excelfiles zu den Daten (Schleife) 
# for testing file<-inputdataset[1]
for(file in inputdataset){
  myimport <- read_excel(paste0("Data/INKAR_1998_2015/",file), skip = 1, sheet = "Daten", col_types = c("text"))
  names(myimport)[1] <- "Kennziffer"
  myimport[3] <- NULL
  myimport[2] <- NULL
  myimport <- myimport %>% gather(key = "Jahr", value = "Value" , -"Kennziffer", convert=T, na.rm = T) %>%
    mutate(Kennziffer=as.numeric(as.character(Kennziffer)), Value=as.numeric(Value)) 
  names(myimport)[3] <- unlist(strsplit(unlist(strsplit(file,"_"))[2],"[.]"))[1]
  Basedata <- full_join(Basedata,myimport,by=c("Kennziffer","Jahr"))
}
# Schleife fÃ¼r jedes Excel-File
# 1. Einlesen der Exceldatei; jeweils das Sheet "Daten"; erste Zeile wird geskippt,  die Daten werden als Text eingelesen
# 2. fÃ¼r die erste Spalte wird die Kennziffer importiert; fÃ¼r die zweite und dritte Spalte nichts
# 3. die Daten werde reshaped, um die Jahresinfos im langen Format zu speichern; convert konvertiert das Datenformat automatisch;
# rm.na entfert missing value Zeilen; -"Kennziffer" sorgt dafÃ¼r, dass die Variable Kennziffer nicht doppelt erzeugt wird
# 4. mutate definiert die Variablentypen 
# 5. von innen nach auÃen 
# 5.1 das innere strsplit(file, "_") teilt den Filenamen inkl. Dateiendung beim "_"
# 5.2 das innerste unlist generiert einen Vektor mit den Elementen aus dem strsplit
# 5.3 das Ã¤uÃere strsplit das zweite Vektorelement beim ".", sodass nur noch der Variablenname Ã¼brig bleibt
# 5.4 das Ã¤uÃere unlist weist auf das erste Vektorelement 
# 5.5 names(import)[3] nimmt dieses Vektorelement als Variablennamen fÃ¼r die dritte Spalte
# 6. jedes file der Schleife wird an Basedata gejoint Ã¼ber Kennziffer und Jahr; full_join Ã¼bernimmt dabei jede Zeile und Spalte jeder Seite,
# auch wenn die Werte auf einer Seite missing enthalten

rm(inputdataset) 


# Liste der Indikatoren erstellen
listofdeterminants <- names(Basedata)[3:length(Basedata)]

# Regionale Tiefe der Indikatoren 
ind_level <- c("Gemeindeverband","Gemeindeverband","Kreis", "Kreis", "Kreis", "Kreis", "Kreis", "Gemeinde", "Kreis", "Kreis")
level_table <- cbind(listofdeterminants,ind_level)
# Tabelle der Indikatoren mit regionaler Tiefe
knitr::kable(level_table)
```



listofdeterminants                ind_level       
--------------------------------  ----------------
Arbeitslosigkeit                  Gemeindeverband 
Beschaeftigtenquote               Gemeindeverband 
Bruttoverdienst                   Kreis           
BeschaeftigtemitakadAbschluss     Kreis           
BeschaeftigteohneAbschluss        Kreis           
SchulabgaengermitHochschulreife   Kreis           
SchulabgaengerohneAbschluss       Kreis           
Einkommenssteuer                  Gemeinde        
Haushaltseinkommen                Kreis           
Schuldnerquote                    Kreis           

```r
# Datensatz für die Gemeindeverbandsebene generieren
Basedata_Gemeindeverbandsebene <- Basedata %>% dplyr::select(Kennziffer,Jahr,Arbeitslosigkeit,Beschaeftigtenquote,Einkommenssteuer) %>%   
  gather(key,value,3:5) %>% filter(!is.na(value)) %>% spread(key,value) %>% filter(Jahr>=1998) %>% rename("Gemeindeverband"=Kennziffer)
# Pipes:  1. Auswahl der Variablen 
#         2. Reshape der Daten wide nach long      
#         3. Auswahl von Non-Missing 
#         4. Reshape von long nach wide 
#         5. Auswahl der Daten Jahr>=1998
#         6. Umbenennung der Kennziffervariable

# Datensatz fÃ¼r die Kreisebene generieren 
Basedata_Kreisebene <- Basedata %>% select(krs15=Kennziffer,Jahr,listofdeterminants) %>% 
  select(-Arbeitslosigkeit,-Einkommenssteuer,-Beschaeftigtenquote) %>% rename(Kreis=krs15)
# Pipes:  1. neben der Kennziffer, die einen anderen Namen bekommt wird das Jahr und die Variablenliste ausgewÃ¤hlt
#         2. drei Variablen werden aus der Auswahl ausgeschlossen
#         3. die Kreisvariable wird in Kreis umbenannt, weil im nÃ¤chsten Schritt Kreisinfos an die Gemeinden angespielt werden

# Join different levels
# Nun werden die Daten bezogen auf die Ebenen gemergt
Workfile <- as.data.frame(expand.grid("Kennziffer"=Gemeinden_INKAR %>% pull(Kennziffer),"Jahr"=seq(min(Basedata$Jahr):max(Basedata$Jahr)) + min(Basedata$Jahr)-1)) %>%
   mutate(Kreiskennziffer=floor(as.numeric(Kennziffer)/1000)) %>% as_tibble() %>%
   left_join(. , Gemeinden_INKAR,by=c("Kennziffer"))  %>%
   select(Gemeindekennziffer=Kennziffer,Kreis=Kreiskennziffer,Gemeindeverband="Kennziffer Gemeindeverband",Jahr,Bevoelkerung=`Bevölkerung 31.12.2015`) %>% 
      arrange(Gemeindekennziffer,Jahr) %>% # Join Metadata
   left_join(. , Basedata_Kreisebene,by=c("Kreis","Jahr")) %>% # Hier wird Ã¼ber Kreis gematched
   left_join(. , Basedata_Gemeindeverbandsebene,by=c("Gemeindeverband","Jahr")) %>%  # Join Indicators for Level: Gemeindeverband 
   filter(Jahr>=1998)

# als erstes wird ein data.frame erzeugt (Workfile); der alle Gemeindewellen (1998-201x) in den Zeilen stehen hat
# 1. expand.grid erzeugt ein tibble mit allen Kombinationen von Kennziffern und Jahren
#     pull erzeugt einen Vektor fÃ¼r die Variablenwerte von Kennziffer aus dem Datensatz
#     + min(...) wird zu der Sequenz von Jahren aus dem Basedata addiert (1 bis X) damit auch Jahreswerte weitergeben werden
# 2. mutate generiert eine Kreiskennziffer
# 3. as.tibble erzeugt einen tibble, damit left_join genutzt werden kann
# 4. erstes left_join spielt die Gemeindedaten Ã¼ber Kennziffer an, das geht so, weil Gemeinden_INKAR als tibble gespeichert ist
# 5. select, wählt die inhaltlichen Variablen aus, und Ã¤ndert die Variablennamen; 
# 6. arrange im select sortiert nach Gemeindekennziffer und Jahr
# 7. zweites left_join spielt die Daten der Kreisebene via Kreis und Jahr an
# 8. drittes left_join spielt die Daten der Gemeindeverbandsebene via Gemeindeverband und Jahr an
# Notiz: . in den Befehlen bezieht sich auf den tibble bzw. data.frame der in der Pipe bearbeitet wird

# Stata-Datensatz rausschreiben
write_dta(Workfile, paste0("Outfiles/2019/Stata/workfile.dta"))

# Ende Generierung Basisdatensatz
```



## III.Imputation fehlender Werte
## IV. Faktorenanalyse (Hauptkomponentenanalyse) inklusive Generierung der Faktorscores
## V.  Datenexport - Erstellung der DatensÃ¤tze 







## Beispiel-Code Plots einbetten

You can also embed plots, for example:

![](GISD_Generate_2015_revision2020_files/figure-html/pressure-1.png)<!-- -->

Note that the `echo = FALSE` parameter was added to the code chunk to prevent printing of the R code that generated the plot.