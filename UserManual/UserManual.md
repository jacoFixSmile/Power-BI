# User manual
- [User manual](#user-manual)
  * [1. Database en schema's aanmaken](#1-database-en-schema-s-aanmaken)
  * [2. Data importeren](#2-data-importeren)
    + [2.1 Datasource](#21-datasource)
    + [2.2 Destination](#22-destination)
    + [2.3 Uitzonderingen](#23-uitzonderingen)
  * [3. Data Cleansing](#3-data-cleansing)
      - [3.1 Cleansed tabellen aanmaken](#31-cleansed-tabellen-aanmaken)
      - [3.2 Stored Procedures](#32-stored-procedures)
        * [3.2.1 Stored Procedures Aanmaken](#321-stored-procedures-aanmaken)
        * [3.2.2 Stored Procedures Uitvoeren](#322-stored-procedures-uitvoeren)
  * [4. Datawarehouse](#4-datawarehouse)
    + [4.1 Tabellen aanmaken](#41-tabellen-aanmaken)
    + [4.2 Data Importeren](#42-data-importeren)

## 1. Database en schema's aanmaken
We maken gebruik van 3 database schema's: RAW, ARCHIVE en CLEANSED. Om de database met de bijhorende shema's te creëren voer je de volgende sql code uit in `Microsoft SQL Server Management Studio:`
```SQL
CREATE DATABASE VisionAirport_OLTP;
GO
USE VisionAirport_OLTP;
EXEC ('CREATE SCHEMA RAW;');
EXEC ('CREATE SCHEMA "ARCHIVE";');
EXEC ('CREATE SCHEMA "CLEANSED";');
GO
```

## 2. Data importeren
De data die we gaan importeren is afkomstig van een aantal flat files(.txt en .csv). Om deze te importeren maak je gebruik van de `SQL Import and Export Wizard:`

<img src="./pictures/ImportExportWizard.png" height="500">

### 2.1 Datasource
Aangezien we gebruik maken van flat files duid je dit aan bij de datasource optie:

<img src="./pictures/DatasourceOption.png">

Na het selecteren van het type van de datasource selecteer je de file onder `File name`. Dit doe je door op de `Browse knop` te drukken en je gewenste file te selecteren. U hoeft verder niets aan te passen aan de kolommen dus u kan gewoon op next drukken tot u aankomt bij `Choose a destination`. 

### 2.2 Destination
Bij de destination combobox scroll je naar onder en selecteert u `Microsoft OLE DB Driver for SQL Server` en als Authentication gebruikt u `Windows Authentication`. Indien u liever gebruik maakt van SQL Authentication kan u dit doen door de username en password in te vullen van uw SQL server. Bij de database selecteerd u de `VisionAirport_OLTP` database die u eerder gecreëerd heeft. 

<img src="./pictures/ChooseDestination.png">

Dan drukt u op next, nu bent u aangekomen bij de `Tables en Views`. Hier zult u het database schema moeten aanpassen van `dbo` naar het database schema `RAW` dat u eerder heeft aangemaakt. Dit doet u door bij destination tussen de `[]` de `dbo` naar `RAW` te veranderen zoals u ziet in onderstaande afbeelding:

<img src="./pictures/ChangeSchema.png">

In principe kan u nu gewoon op next blijven drukken en het process afronden. Bij de meeste files zul je verder niets moeten aanpassen aan de kolommen etc, sommige datasources zullen een error geven, dit zal u ervaren bij de volgende files: `Export_luchthavens.txt` en `Export_vielgtuigtype.txt`. 

### 2.3 Uitzonderingen

Aangezien `Export_luchthavens.txt` en `Export_vielgtuigtype.txt` voor errors zullen zorgen moet u de structuur van deze tabellen aanpassen.
Bij `Export_luchthavens.txt` heeft 2 problemen: `er zijn 2 tabellen met dezelfde naam` en `de kolom Airport wordt getruncate`. Om dit op te lossen doet u het volgende:

Laten we beginnen bij de errors op te lossen van de het bestand `Export_luchthavens`. Bij het kiezen van een datasource, voor dat u op next drukt, moet u eerst naar de `Advanced tab` gaan die zich links in het venster bevindt.

<img src="./pictures/DatasourceAdvanced.png">

Aangezien er 2 kollomen zijn met dezelfde naam, `TZ` en `Tz`, moeten we bij 1 van de 2 kollomen de naam aanpassen. Vervang bij de kolom `Tz` de naam naar `Tz_LowerCase` zoals u ziet op de volgende afbeelding:

<img src="./pictures/DoubleNameSolution.png">

Om de kolom Airports die getruncate is op te lossen vervangt u het datatype van de kolom naar een `Text stream`. Als de varchar niet als `size max` heeft, verander dit dan in ‘Edit Mappings’

<img src="./pictures/TruncateSolution.png">

Bij de file `Export_vielgtuigtype.txt` is het enige probleem dat de kolom `Type` wordt getruncate. Dit lost u op op dezeldfde manier zoals u hierboven bij de kolom `Airport` gedaan hebt.

## 3. Data Cleansing

#### 3.1 Cleansed tabellen aanmaken
Alle tabellen worden vernoemd met de prefix `tbl` en het `Export_` gedeelte wordt weggehaald. Dus `Export_weer` wordt dan `tblWeer`
Voor het creëeren van de tabellen kan u het volgende SQL script uitvoeren:

<details>
<summary>SQL script om de cleansed tabellen aan te maken</summary>
<p>

```SQL
use VisionAirport_OLTP;

-- CLEANSED TABLES
CREATE TABLE CLEANSED.vliegtuigtype
(
	IATA CHAR(3) NOT NULL,
	ICAO VARCHAR(4) NULL,
	Merk VARCHAR(50) NULL,
	Type VARCHAR(max) NOT NULL,
	Wake VARCHAR(3) NULL,
	Cat VARCHAR(10) NULL,
	Capaciteit SMALLINT NULL,
	Vracht SMALLINT NULL,
	PRIMARY KEY(IATA)
);

CREATE TABLE CLEANSED.vliegtuig
(
	VliegtuigId INT NOT NULL IDENTITY,
	Airlinecode VARCHAR(3) NOT NULL,
	Vliegtuigcode VARCHAR(10) NOT NULL,
	Vliegtuigtype CHAR(3) NOT NULL,
	Bouwjaar SMALLINT NOT NULL,
	PRIMARY KEY(VliegtuigId),
	FOREIGN KEY(Vliegtuigtype) REFERENCES CLEANSED.vliegtuigtype(IATA)
);

CREATE TABLE CLEANSED.planning
(
	PlanningId INT NOT NULL IDENTITY,
	Vluchtnr VARCHAR(10) NOT NULL,
	Airlinecode VARCHAR(3) NOT NULL,
	Destcode CHAR(3) NOT NULL,
	Planterminal CHAR NULL,
	Plangate CHAR(2) NULL,
	Plantijd TIME NULL,
	PRIMARY KEY(PlanningId),
);

CREATE TABLE CLEANSED.vlucht
(
	Vluchtid INT NOT NULL,
	Vluchtnr VARCHAR(10) NULL,
	Airlinecode VARCHAR(3) NULL,
	Destcode CHAR(3) NOT NULL,
	Vliegtuigcode VARCHAR(10) NOT NULL,
	Datum DATE NOT NULL,
	PRIMARY KEY(Vluchtid),
);

CREATE TABLE CLEANSED.banen
(
	Baannummer INT NOT NULL,
	Code VARCHAR(10) NOT NULL,
	Naam VARCHAR(50) NOT NULL,
	Lengte SMALLINT NOT NULL,
	PRIMARY KEY(Baannummer)
);

CREATE TABLE CLEANSED.aankomst
(
	AankomstId INT NOT NULL IDENTITY,
	Vluchtid INT NOT NULL,
	Vliegtuigcode VARCHAR(10) NOT NULL,
	Terminal CHAR NULL,
	Gate CHAR(2) NULL,
	Baan INT NULL,
	Bezetting SMALLINT NULL,
	Vracht SMALLINT NULL,
	Aankomsttijd DATETIME NULL,
	PRIMARY KEY(AankomstId),
	FOREIGN KEY(Vluchtid) REFERENCES CLEANSED.vlucht(Vluchtid),
	FOREIGN KEY(Baan) REFERENCES CLEANSED.banen(Baannummer)
);

CREATE TABLE CLEANSED.vertrek
(
	VertrekId INT NOT NULL IDENTITY,
	Vluchtid INT NOT NULL,
	Vliegtuigcode VARCHAR(10) NOT NULL,
	Terminal CHAR NULL,
	Gate CHAR(2) NULL,
	Baan INT NULL,
	Bezetting SMALLINT NULL,
	Vracht SMALLINT NULL,
	Aankomsttijd DATETIME NULL,
	PRIMARY KEY(VertrekId),
	FOREIGN KEY(Vluchtid) REFERENCES CLEANSED.vlucht(Vluchtid),
	FOREIGN KEY(Baan) REFERENCES CLEANSED.banen(Baannummer)
);

CREATE TABLE CLEANSED.klant
(
	KlantId INT NOT NULL IDENTITY,
	Vluchtid INT NOT NULL,
	Operatie DECIMAL(2,1) NULL,
	Faciliteiten DECIMAL(2,1) NULL,
	Shops DECIMAL(2,1) NULL,
	PRIMARY KEY(KlantId),
	FOREIGN KEY(Vluchtid) REFERENCES CLEANSED.vlucht(Vluchtid)
);

CREATE TABLE CLEANSED.luchthavens
(
	LuchthavenId INT NOT NULL IDENTITY,
	Airport VARCHAR(max) NOT NULL,
	City VARCHAR(50) NOT NULL,
	Country VARCHAR(50) NOT NULL,
	IATA VARCHAR(3) NULL,
	ICAO VARCHAR(4) NULL,
	Lat FLOAT NOT NULL,
	Lon FLOAT NOT NULL,
	Alt SMALLINT NOT NULL,
	TZ FLOAT NOT NULL,
	DST CHAR NOT NULL,
	Timezone VARCHAR(50) NOT NULL,
	PRIMARY KEY(LuchthavenId)
);

CREATE TABLE CLEANSED.maatschappijen
(
	MaatschappijId INT NOT NULL IDENTITY,
	Name VARCHAR(50) NOT NULL,
	IATA CHAR(2) NULL,
	ICAO CHAR(3) NULL,
	PRIMARY KEY(MaatschappijId)
);

CREATE TABLE CLEANSED.weer
(
	WeerId INT NOT NULL IDENTITY,
	Datum DATE NOT NULL,
	DDVEC SMALLINT NOT NULL,
	FHVEC SMALLINT NOT NULL,
	FG SMALLINT NOT NULL,
	FHX SMALLINT NOT NULL,
	FHXH SMALLINT NOT NULL,
	FHN SMALLINT NOT NULL,
	FHNH SMALLINT NOT NULL,
	FXX SMALLINT NOT NULL,
	FXXH SMALLINT NOT NULL,
	TG SMALLINT NOT NULL, 
	TN SMALLINT NOT NULL,
	TNH SMALLINT NOT NULL,
	TX SMALLINT NOT NULL, 
	TXH SMALLINT NOT NULL,
	T10N SMALLINT NOT NULL,
	T10NH SMALLINT NOT NULL,
	SQ SMALLINT NOT NULL,
	SP SMALLINT NOT NULL,
	Q SMALLINT NOT NULL,
	DR SMALLINT NOT NULL,
	RH SMALLINT NOT NULL,
	RHX SMALLINT NOT NULL,
	RHXH SMALLINT NOT NULL,
	PG SMALLINT NOT NULL,
	PX SMALLINT NOT NULL,
	PXH SMALLINT NOT NULL,
	PN SMALLINT NOT NULL,
	PNH SMALLINT NOT NULL,
	VVN SMALLINT NOT NULL,
	VVNH SMALLINT NOT NULL,
	VVX SMALLINT NOT NULL,
	VVXH SMALLINT NOT NULL,
	NG SMALLINT NOT NULL,
	UG SMALLINT NOT NULL,
	UX SMALLINT NOT NULL,
	UXH SMALLINT NOT NULL,
	UN SMALLINT NOT NULL,
	UNH SMALLINT NOT NULL,
	EV2 SMALLINT NOT NULL,
	PRIMARY KEY(WeerId)
);

-- ARCHIVE TABLES
CREATE TABLE ARCHIVE.vliegtuigtype
(
	IATA VARCHAR(50) NULL,
	ICAO VARCHAR(50) NULL,
	Merk VARCHAR(50) NULL,
	Type VARCHAR(max) NULL,
	Wake VARCHAR(50) NULL,
	Cat VARCHAR(50) NULL,
	Capaciteit VARCHAR(50) NULL,
	Vracht VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.vliegtuig
(
	Airlinecode VARCHAR(50) NULL,
	Vliegtuigcode VARCHAR(50) NULL,
	Vliegtuigtype VARCHAR(50) NULL,
	Bouwjaar VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.planning
(
	Vluchtnr VARCHAR(50) NULL,
	Airlinecode VARCHAR(50) NULL,
	Destcode VARCHAR(50) NULL,
	Planterminal VARCHAR(50) NULL,
	Plangate VARCHAR(50) NULL,
	Plantijd VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.vlucht
(
	Vluchtid VARCHAR(50) NULL,
	Vluchtnr VARCHAR(50) NULL,
	Airlinecode VARCHAR(50) NULL,
	Destcode VARCHAR(50) NULL,
	Vliegtuigcode VARCHAR(50) NULL,
	Datum VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.banen
(
	Baannummer VARCHAR(50) NULL,
	Code VARCHAR(50) NULL,
	Naam VARCHAR(50) NULL,
	Lengte VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.aankomst
(
	Vluchtid VARCHAR(50) NULL,
	Vliegtuigcode VARCHAR(50) NULL,
	Terminal VARCHAR(50) NULL,
	Gate VARCHAR(50) NULL,
	Baan VARCHAR(50) NULL,
	Bezetting VARCHAR(50) NULL,
	Vracht VARCHAR(50) NULL,
	Aankomsttijd VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.vertrek
(
	Vluchtid VARCHAR(50) NULL,
	Vliegtuigcode VARCHAR(50) NULL,
	Terminal VARCHAR(50) NULL,
	Gate VARCHAR(50) NULL,
	Baan VARCHAR(50) NULL,
	Bezetting VARCHAR(50) NULL,
	Vracht VARCHAR(50) NULL,
	Aankomsttijd VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.klant
(
	Vluchtid VARCHAR(50) NULL,
	Operatie VARCHAR(50) NULL,
	Faciliteiten VARCHAR(50) NULL,
	Shops VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.luchthavens
(
	Airport VARCHAR(max) NULL,
	City VARCHAR(50) NULL,
	Country VARCHAR(50) NULL,
	IATA VARCHAR(50) NULL,
	ICAO VARCHAR(50) NULL,
	Lat VARCHAR(50) NULL,
	Lon VARCHAR(50) NULL,
	Alt VARCHAR(50) NULL,
	TZ VARCHAR(50) NULL,
	DST VARCHAR(50) NULL,
	Timezone VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.maatschappijen
(
	Name VARCHAR(50) NULL,
	IATA VARCHAR(50) NULL,
	ICAO VARCHAR(50) NULL
);

CREATE TABLE ARCHIVE.weer
(
	Datum VARCHAR(50) NULL,
	DDVEC VARCHAR(50) NULL,
	FHVEC VARCHAR(50) NULL,
	FG VARCHAR(50) NULL,
	FHX VARCHAR(50) NULL,
	FHXH VARCHAR(50) NULL,
	FHN VARCHAR(50) NULL,
	FHNH VARCHAR(50) NULL,
	FXX VARCHAR(50) NULL,
	FXXH VARCHAR(50) NULL,
	TG VARCHAR(50) NULL, 
	TN VARCHAR(50) NULL,
	TNH VARCHAR(50) NULL,
	TX VARCHAR(50) NULL, 
	TXH VARCHAR(50) NULL,
	T10N VARCHAR(50) NULL,
	T10NH VARCHAR(50) NULL,
	SQ VARCHAR(50) NULL,
	SP VARCHAR(50) NULL,
	Q VARCHAR(50) NULL,
	DR VARCHAR(50) NULL,
	RH VARCHAR(50) NULL,
	RHX VARCHAR(50) NULL,
	RHXH VARCHAR(50) NULL,
	PG VARCHAR(50) NULL,
	PX VARCHAR(50) NULL,
	PXH VARCHAR(50) NULL,
	PN VARCHAR(50) NULL,
	PNH VARCHAR(50) NULL,
	VVN VARCHAR(50) NULL,
	VVNH VARCHAR(50) NULL,
	VVX VARCHAR(50) NULL,
	VVXH VARCHAR(50) NULL,
	NG VARCHAR(50) NULL,
	UG VARCHAR(50) NULL,
	UX VARCHAR(50) NULL,
	UXH VARCHAR(50) NULL,
	UN VARCHAR(50) NULL,
	UNH VARCHAR(50) NULL,
	EV2 VARCHAR(50) NULL
);
```

</p>
</details>  

#### 3.2 Stored Procedures
##### 3.2.1 Stored Procedures Aanmaken
Aan de hand van stored procedures wordt de data gecleansed. De data wordt gehaald uit de RAW tabellen. De `vuile data` wordt eruit gefiltered met behulp van regex en wordt dan in de ARCHIVE tabellen geplaatst. Nu is de data gecleansed en wordt het dus in de overeenkomstige CLEANSED tabellen geplaatst.

Om de stored procedures zo universeel mogelijk te maken, hebben we gebruik gemaak van Regex. Deze zijn beter in het filteren van de data dan de ingebouwde expresions van SQL, die zeer beperkt zijn. hiervoor zijn we gebruik gaan maken van een zelf geschreven C# CLR functie. 
Voer daarvoor op je SQL instantie uit 

```SQL
EXEC sp_configure 'show advanced options', 1
    RECONFIGURE
	EXEC sp_configure 'clr strict security', 0
EXEC sp_configure 'clr enabled';  
EXEC sp_configure 'clr enabled' , '1';  
RECONFIGURE;  
```
Vervolgens moet je de CLR C# Functie nog uitvoeren. Open hiervoor `RegularExpressionSQL` en klik hier op `RegularExpressionSQL.sln`, dit opent de code voor de regex. Druk nu vanboven op start en selecteer dan de server waar `VisionAirport_OLTP` opstaat en ook de database om de functie eraan toe te voegen. Als dit een error geeft, moet je de connection string nog veranderen. Klik in de solution explorer op `Properties`, ga naar `Debug` en scroll naar beneden. Onder `Target Connection string` drukt u op edit. Selecteer weer de correcte server en  `VisionAirport_OLTP` als database, druk vervolgens op ok. Voer nu opnieuw uit door op Start te drukken.

Om de stored procedures te creëeren voer je de volgende SQL querries uit:

[comment]: <> (Stored Prodedure Banen)
<details>
<summary>Stored Procedure Banen</summary>
<p>

```SQL
create PROCEDURE sp_banen
AS

INSERT INTO VisionAirport_OLTP.CLEANSED.banen
SELECT [Baannummer]
      ,[Code]
      ,[Naam]
      ,[Lengte]
  FROM [VisionAirport_OLTP].[RAW].[export_banen] where 
  dbo.EvaluateRegex('[0-9]+',[Baannummer])=1 AND 
  dbo.EvaluateRegex('[0-9A-Z-]+',[Code])=1 AND
  dbo.EvaluateRegex('[0-9A-Za-z ]+',[Naam])=1 AND
   dbo.EvaluateRegex('[0-9]+',[Lengte])=1

   INSERT INTO VisionAirport_OLTP.[ARCHIVE].banen
SELECT [Baannummer]
      ,[Code]
      ,[Naam]
      ,[Lengte]
  FROM [VisionAirport_OLTP].[RAW].[export_banen] where 
  dbo.EvaluateRegex('[0-9]+',[Baannummer])+
  dbo.EvaluateRegex('[0-9A-Z-]+',[Code])+
  dbo.EvaluateRegex('[0-9A-Za-z ]+',[Naam])+
   dbo.EvaluateRegex('[0-9]+',[Lengte])!=4
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure VliegtuigType)
<details>
<summary>Stored Procedure VliegtuigType</summary>
<p>

```SQL
CREATE PROCEDURE sp_vliegtuigtype
AS
BEGIN

INSERT INTO CLEANSED.vliegtuigtype (IATA, ICAO, Merk, Type, Wake, Cat, Capaciteit, Vracht)
SELECT
	*
FROM
	RAW.export_vliegtuigtype
WHERE
	IATA NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND
	ICAO NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND
	Merk NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND
	Type NOT LIKE '%[,!?;;\@#ｮ%ﾐﾞ恁嚏梹歉%'
	AND
	Wake NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND
	Cat NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND 
	Capaciteit NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	AND
	Vracht NOT LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%';	

UPDATE CLEANSED.vliegtuigtype 
	SET 
		ICAO = CASE
	WHEN ICAO = 'n/a' THEN NULL
	WHEN ICAO = '' THEN NULL
	ELSE ICAO
	END,
		Merk = CASE
	WHEN Merk = '' THEN NULL
	ELSE Merk
	END,
		Wake = CASE
	WHEN Wake = 'n/a' THEN NULL
	WHEN Wake = '' THEN NULL
	ELSE Wake
	END,
		Cat = CASE
	WHEN Cat = '' THEN NULL
	ELSE Cat
	END,
		Capaciteit = CASE
	WHEN Capaciteit = '' THEN NULL
	ELSE Capaciteit
	END,
		Vracht = CASE
	WHEN Vracht = '' THEN NULL
	ELSE Vracht
	END;
	
INSERT INTO ARCHIVE.vliegtuigtype (IATA, ICAO, Merk, Type, Wake, Cat, Capaciteit, Vracht)
SELECT
	*
FROM
	RAW.export_vliegtuigtype
WHERE
	IATA LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR
	ICAO LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR
	Merk LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR
	Type LIKE '%[,!?;;\@#ｮ%ﾐﾞ恁嚏梹歉%'
	OR
	Wake LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR
	Cat LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR 
	Capaciteit LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%'
	OR
	Vracht LIKE '%[,!?;;\@#ｮ.%ﾐﾞ恁嚏梹歉%';	

END
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Vliegtuig)
<details>
<summary>Stored Procedure Vliegtuig</summary>
<p>

```SQL
CREATE PROCEDURE sp_Vliegtuig
AS
insert into CLEANSED.vliegtuig select * from RAW.export_vliegtuig
UPDATE CLEANSED.vliegtuig 
SET Airlinecode = NULL 
WHERE Airlinecode = '-'
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Weer)
<details>
<summary>Stored Procedure Weer</summary>
<p>

```SQL
CREATE PROCEDURE sp_Weer
AS
insert into CLEANSED.WEER
select * from RAW.export_weer
where DATUM LIKE '[1-2][0-9][0-9][0-9]-[0-1][0-9]-[0-3][0-9]' 
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Luchthavens)
<details>
<summary>Stored Procedure Luchthavens</summary>
<p>

```SQL
CREATE PROCEDURE sp_luchthavens
AS
BEGIN

INSERT INTO CLEANSED.luchthavens (Airport, City, Country, IATA, ICAO, Lat, Lon, Alt, TZ, DST, Timezone)
SELECT
	*
FROM
	RAW.export_luchthavens
WHERE
	Airport NOT LIKE '%[,!?;;\@#о╨▐■ЬМЪКЮОЯ]%'
	AND
	City NOT LIKE '%[,!?;;\@#о%╨▐■ЬМЪКЮОЯ]%'
	AND
	Country NOT LIKE '%[,!?;;\@#о%.╨▐■ЬМЪКЮОЯ]%'
	AND
	IATA NOT LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	AND
	ICAO NOT LIKE '%[,!?;;@#%о.╨▐■ЬМЪКЮОЯ]%'
	AND
	Lat NOT LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	AND 
	Lon NOT LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	AND
	Alt NOT LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	AND
	TZ NOT LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	AND
	DST NOT LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	AND
	Timezone NOT LIKE '%[,!?;;@#%о.╨▐■ЬМЪКЮОЯ]%'; 

UPDATE CLEANSED.luchthavens 
	SET 
		IATA = CASE 
	WHEN 
		IATA = '' THEN NULL
	ELSE 
		IATA
	END, 
		ICAO = CASE
	WHEN 
		ICAO = '\N' THEN NULL
	ELSE
		ICAO
	END;
	
INSERT INTO ARCHIVE.luchthavens (Airport, City, Country, IATA, ICAO, Lat, Lon, Alt, TZ, DST, Timezone)
SELECT
	*
FROM
	RAW.export_luchthavens
WHERE
	Airport LIKE '%[,!?;;\@#о╨▐■ЬМЪКЮОЯ]%'
	OR
	City LIKE '%[,!?;;\@#о%╨▐■ЬМЪКЮОЯ]%'
	OR
	Country LIKE '%[,!?;;\@#о%.╨▐■ЬМЪКЮОЯ]%'
	OR
	IATA LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	OR
	ICAO LIKE '%[,!?;;@#%о.╨▐■ЬМЪКЮОЯ]%'
	OR
	Lat LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	OR 
	Lon LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	OR
	Alt LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	OR
	TZ LIKE '%[,!?;;@#%\о╨▐■ЬМЪКЮОЯ]%'
	OR
	DST LIKE '%[,!?;;@#%\о.╨▐■ЬМЪКЮОЯ]%'
	OR
	Timezone LIKE '%[,!?;;@#%о.╨▐■ЬМЪКЮОЯ]%'; 

END
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Maatschappijen)
<details>
<summary>Stored Procedure Maatschappijen</summary>
<p>

```SQL
create PROCEDURE sp_maatschappijen
AS

INSERT INTO VisionAirport_OLTP.CLEANSED.[maatschappijen]
SELECT [Name]
      ,[IATA]
      ,replace(replace([ICAO],'\N',''),'N/A','') as  ICAO
  FROM [VisionAirport_OLTP].[RAW].[export_maatschappijen] where 
  dbo.EvaluateRegex('[A-Za-z0-9& ]+',[Name])=1 AND 
  dbo.EvaluateRegex('[0-9A-Z ]+',[IATA])=1 AND
  dbo.EvaluateRegex('[0-9A-Z ]+',ICAO)=1 

  
INSERT INTO VisionAirport_OLTP.[ARCHIVE].[maatschappijen]
SELECT [Name]
      ,[IATA]
      ,replace(replace([ICAO],'\N',''),'N/A','') as  ICAO
  FROM [VisionAirport_OLTP].[RAW].[export_maatschappijen] where 
  dbo.EvaluateRegex('[A-Za-z0-9& ]+',[Name])+ 
  dbo.EvaluateRegex('[0-9A-Z ]+',[IATA])+
  dbo.EvaluateRegex('[0-9A-Z ]+',ICAO)!=3

  
UPDATE [VisionAirport_OLTP].[CLEANSED].[maatschappijen]
	SET 
		Name = CASE
	WHEN Name = ''  THEN NULL
	ELSE Name
	END,
		IATA = CASE
	WHEN IATA = ''  THEN NULL
	ELSE IATA
	END,
		ICAO = CASE
	WHEN ICAO = ''  THEN NULL
	ELSE ICAO
	END;

GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Vlucht)
<details>
<summary>Stored Procedure Vlucht</summary>
<p>

```SQL
CREATE PROCEDURE sp_vlucht
AS
BEGIN

INSERT INTO CLEANSED.vlucht (Vluchtid, Vluchtnr, Airlinecode, Destcode, Vliegtuigcode, Datum)
SELECT
	*
FROM
	RAW.export_vlucht
WHERE
	Vluchtid NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	AND
	Vluchtnr NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	AND
	Airlinecode NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	AND
	Destcode NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	AND
	Vliegtuigcode NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	AND
	Datum NOT LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'	

UPDATE CLEANSED.vlucht 
	SET 
		Vluchtnr = CASE
	WHEN Vluchtnr = '' THEN NULL
	ELSE Vluchtnr
	END,
		Airlinecode = CASE
	WHEN Airlinecode = '-' THEN NULL
	ELSE Airlinecode
	END;
	
INSERT INTO ARCHIVE.vlucht (Vluchtid, Vluchtnr, Airlinecode, Destcode, Vliegtuigcode, Datum)
SELECT
	*
FROM
	RAW.export_vlucht
WHERE
	Vluchtid LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	OR
	Vluchtnr LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	OR
	Airlinecode LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	OR
	Destcode LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	OR
	Vliegtuigcode LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%'
	OR
	Datum LIKE '%[,!?;;\@#®.%ÐÞþœŒšŠžŽŸ]%';

END
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Vertrek)
<details>
<summary>Stored Procedure Vertrek</summary>
<p>

```SQL
CREATE PROCEDURE sp_vertrek
AS
BEGIN 

insert into [CLEANSED].[vertrek] (Vluchtid, Vliegtuigcode, Terminal, Gate, Baan, Bezetting, Vracht, Aankomsttijd)
select 
	*
from [VisionAirport_OLTP].RAW.[export_vertrek] where Vluchtid NOT LIKE '%[+]%' AND baan IN (SELECT baannummer FROM [CLEANSED].[banen])

UPDATE [CLEANSED].[vertrek]
	SET 
		[Gate] = CASE
	WHEN [Gate] = ''  THEN NULL
	ELSE [Gate]
	END,
		[Bezetting] = CASE
	WHEN [Bezetting] = ''  THEN NULL
	ELSE [Bezetting]
	END,
		[Vracht] = CASE
	WHEN [Vracht] = ''  THEN NULL
	ELSE [Vracht]
	END,
		[Aankomsttijd] = CASE
	WHEN [Aankomsttijd] = ''  THEN NULL
	ELSE [Aankomsttijd]
	END;

insert into [ARCHIVE].[vertrek] (Vluchtid, Vliegtuigcode, Terminal, Gate, Baan, Bezetting, Vracht, Aankomsttijd)
select 
	*
from [VisionAirport_OLTP].RAW.[export_vertrek] where Vluchtid LIKE '%[+]%' OR baan NOT IN (SELECT baannummer FROM [CLEANSED].[banen])

END
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Aankomst)
<details>
<summary>Stored Procedure Aankomst</summary>
<p>

```SQL
CREATE PROCEDURE sp_aankomst
AS
BEGIN 

insert into [VisionAirport_OLTP].[CLEANSED].[aankomst] (Vluchtid, Vliegtuigcode, Terminal, Gate, Baan, Bezetting, Vracht, Aankomsttijd)
select 
	*
from [VisionAirport_OLTP].RAW.[export_aankomst] where Vluchtid NOT LIKE '%[+]%' AND baan IN (SELECT baannummer FROM [VisionAirport_OLTP].[CLEANSED].[banen])

UPDATE [VisionAirport_OLTP].[CLEANSED].[aankomst]
	SET 
		[Gate] = CASE
	WHEN [Gate] = ''  THEN NULL
	ELSE [Gate]
	END,
		[Bezetting] = CASE
	WHEN [Bezetting] = ''  THEN NULL
	ELSE [Bezetting]
	END,
		[Vracht] = CASE
	WHEN [Vracht] = ''  THEN NULL
	ELSE [Vracht]
	END,
		[Aankomsttijd] = CASE
	WHEN [Aankomsttijd] = ''  THEN NULL
	ELSE [Aankomsttijd]
	END;

insert into [VisionAirport_OLTP].[ARCHIVE].[aankomst] (Vluchtid, Vliegtuigcode, Terminal, Gate, Baan, Bezetting, Vracht, Aankomsttijd)
select 
	*
from [VisionAirport_OLTP].RAW.[export_aankomst] where Vluchtid LIKE '%[+]%' OR baan NOT IN (SELECT baannummer FROM [VisionAirport_OLTP].[CLEANSED].[banen])

END
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Klant)
<details>
<summary>Stored Procedure Klant</summary>
<p>

```SQL
alter PROCEDURE sp_klant
AS

INSERT INTO VisionAirport_OLTP.CLEANSED.klant
SELECT [Vluchtid]
      ,(CONVERT(DECIMAL(2, 1), ISNULL(NULLIF([Operatie], ''), '0')))
      ,(CONVERT(DECIMAL(2, 1), ISNULL(NULLIF([Operatie], ''), '0')))
      ,(CONVERT(DECIMAL(2, 1), ISNULL(NULLIF([Shops], ''), '0')))
  FROM [VisionAirport_OLTP].[RAW].[export_klant] where 
  dbo.EvaluateRegex('[0-9]+',[Vluchtid])=1 AND 
  dbo.EvaluateRegex('[0-9.]+',[Operatie])=1 AND
  dbo.EvaluateRegex('[0-9.]+',[Faciliteiten])=1 AND
   dbo.EvaluateRegex('[0-9.]+',[Shops])=1

INSERT INTO VisionAirport_OLTP.[ARCHIVE].klant
SELECT [Vluchtid]
      ,[Operatie]
      ,[Faciliteiten]
      ,[Shops]
  FROM [VisionAirport_OLTP].[RAW].[export_klant] where 
  dbo.EvaluateRegex('[0-9]+',[Vluchtid])+
	  dbo.EvaluateRegex('[0-9.]+',[Operatie])+
  dbo.EvaluateRegex('[0-9.]+',[Faciliteiten])+
   dbo.EvaluateRegex('[0-9.]+',[Shops])!=4
GO
```

</p>
</details> 
<br/>

[comment]: <> (Stored Prodedure Planning)
<details>
<summary>Stored Procedure Planning</summary>
<p>

```SQL
CREATE PROCEDURE sp_Planning
AS
insert into CLEANSED.planning select * from dbo.export_planning;
UPDATE CLEANSED.planning 
SET Planterminal = NULL 
WHERE Planterminal = ''
UPDATE CLEANSED.planning 
SET Plangate = NULL 
WHERE Plangate = ''
UPDATE CLEANSED.planning 
SET Plantijd = NULL 
WHERE Plantijd = ''
GO
```

</p>
</details> 
<br/>


##### 3.2.2 Stored Procedures Uitvoeren
Na het aanmaken van de stored procedures moeten ze enkel nog uitgevoerd worden. Dat doet u door het uitvoeren van het volgende SQL script:

```SQL
USE VisionAirport_OLTP;

EXEC sp_banen;
EXEC sp_vliegtuigtype;
EXEC sp_Vliegtuig;
EXEC sp_Weer;
EXEC sp_Luchthavens;
EXEC sp_Maatschappijen;
EXEC sp_Vluch;
EXEC sp_Vertrek;
EXEC sp_Aankomst;
EXEC sp_Klant;
EXEC sp_Planning;
GO
```


## 4. Datawarehouse
### 4.1 Tabellen aanmaken
Nu dat u al de data heeft geïmporteerd en alle vuile data uit de tabellen gehaald heeft, kan u beginnen met het opstellen van het datawarehouse. Het aanmaken van deze database met bijhorende tabellen doet u aan de hand van de volgende SQL scripts:
```SQL
CREATE DATABASE VisionAirport_DWH;
```

<details>
<summary>SQL script om de DWH tabellen aan te maken</summary>
<p>

```SQL
use VisionAirport_DWH;
CREATE TABLE dim_planning
(
	PlanningID INT,
	Planterminal CHAR NULL,
	Plangate CHAR(2) NULL,
	Plantijd TIME Null,
	vluchtnr varchar(50),
	PRIMARY KEY(PlanningID),
);
CREATE TABLE dim_banen
(
	Baannr INT,
	Code VARCHAR(10) NOT NULL,
	Naam VARCHAR(50) NOT NULL,
	Lengte SMALLINT NOT NULL,
	PRIMARY KEY(Baannr)
);

CREATE TABLE dim_vliegtuig
(
	VliegtuigID int IDENTITY,
	VliegtuigCode varchar(10),
	Airlinecode VARCHAR(3),
	Bouwjaar SMALLINT,
	Merk VARCHAR(50),
	Type VARCHAR(MAX),
	Wake VARCHAR(3),
	Catogrie VARCHAR(10),
	Capacititeit SMALLINT,
	Vracht SMALLINT,
	IATA CHAR(3),
	ICAO VARCHAR(4),
	PRIMARY KEY(VliegtuigID)
);

CREATE TABLE dim_datum
(
	DateID INT IDENTITY,
	Datum DATE,
	WeekDagNummer SMALLINT,
	MaandNummer SMALLINT,
	Jaar SMALLINT,
	WeekDag Char(2),
	Maand Char(2),
	KalenderDagNummer SMALLINT,
	KalenderWeekNummer SMALLINT,
	GemiddeldeWindsnelheid SMALLINT,
	GemiddeledeTempratuur SMALLINT,
	GemiddeldeZonneschijnduur SMALLINT,
	GemiddeldeNeerslagduur SMALLINT,
	GemiddeldeZicht SMALLINT,
	GemiddeldeBewolking SMALLINT,
	PRIMARY KEY (DateID)
);

CREATE TABLE dim_maatschappijen
(
	MaatschapijID INT,
	Name VARCHAR(50),
	IATA CHAR(2),
	ICAO CHAR(3),
	PRIMARY KEY(MaatschapijID)
);
CREATE TABLE dim_luchthaven
(
LuchthavenID INT,
Naam VARCHAR(max),
City VARCHAR(50),
Country VARCHAR(50),
IATA VARCHAR(3),
ICAO VARCHAR(4),
Latitude FLOAT,
Longitude FLOAT,
Timezone VARCHAR(50),
PRIMARY KEY(LuchthavenID)
);

CREATE TABLE dim_gate
(
GateID INT IDENTITY,
Gate CHAR(2),
Terminal CHAR,
PRIMARY KEY (GateID)
);

create table dim_klant
(
KlantId int IDENTITY,
Operatie decimal(2,1),
Faciliteiten decimal(2,1),
Shops decimal(2,1),
vluchtnr varchar(50),
PRIMARY KEY(KlantId)
);

CREATE TABLE fact_vlucht
(
VluchtID INT,
VliegtuigID INT,
MaatschapijID INT,
LuchthavenID INT,
PlanningID INT,
DateID INT,
GateID INT,
KlantID INT,
Baannr INT,
Vluchtnr VARCHAR(10),
DestCode CHAR(3),
Bezeting INT,
Tijd TIME,
isAankomst TINYINT,
FOREIGN KEY(VliegtuigID)REFERENCES dim_vliegtuig(VliegtuigID),
FOREIGN KEY(MaatschapijID) REFERENCES dim_maatschappijen(MaatschapijID),
FOREIGN KEY(LuchthavenID) REFERENCES dim_luchthaven(LuchthavenID),
FOREIGN KEY(PlanningID) REFERENCES dim_planning(PlanningID),
FOREIGN KEY(DateID) REFERENCES dim_datum(DATEID),
FOREIGN KEY(GateID) REFERENCES dim_gate(GateID),
FOREIGN KEY(Baannr) REFERENCES dim_banen(Baannr),
FOREIGN KEY(KlantID)REFERENCES dim_klant(KlantId),
PRIMARY KEY(VluchtID)
);
```
</p>
</details> 


#### 4.2 Data Importeren
Voor de data te importeren in een DWH gaan we ETL procedure gerbuiken. Open hiervoor in de ETLVisonAirport map ETLVisionAirport.sln met Visual Studio 2019. Wacht hier tot het programma geladen is en druk dan start. wacht tot alle flows gedaan zijn met overlopen, dit kan je zien doormiddel van het "V" naast elke flow. De data is nu geinporteerd in je DWH. Je kan dit zoveel herhalen als je wilt, hij zal telkens enkel de nieuwe data inladen vanuit de OLTP.   