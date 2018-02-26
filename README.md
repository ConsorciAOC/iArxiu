# iArxiu
Projecte amb tests SoapUI d'exemple d'integració al servei d'iArxiu.

# Configuració

## 1. Projecte SoapUI
###1.1 **pre.propeties**

```
ens=XXX (nom de l'ens)
fons=XXX (nom del fons)
rol=archivists (rol de l'usuari que fa l'ingrés)
url=www.preproduccio.iarxiu.eacat.cat (url de l'entorn d'iarxiu)
Subject=XXX (nom de l'usuari que fa l'ingrés)
Issuer=SoapUI_Testing (aplicació)
downloadUrl=https://www.preproduccio.iarxiu.eacat.cat/
(variable internes dels tests, no cal editar)
IngestStatusRequest= 
uploadTicket=
uploadservleturl=
packageIdDoc=
```

###1.2 Arrel del projecte
Des de la vista de Project View (clic amb botó dret sobre el projecte importat al SoapUI > Show Projecte View), editar la propietat ```baseDir``` i definir l'arrel del projecte dins del LoadScript (script groovy que s'executa al carregar el projecte).

![SoapUI Project View > Overview > LoadScript](/img/loadscript.PNG?raw=true "SoapUI Project View > Overview > LoadScript")

## 2. Web de referència iArxiu (http://www.preproduccio.iarxiu.eacat.cat/)
###2.1. Plantilles
Els fitxers mets.xml fan ús de plantilles (on es defineixen els vocabularis permesos per construir les metadades). El primer cas d'ús (Expedient_UpDown_XAdES_Detached_Zip) fa ús de la plantilla urn:iarxiu:2.0:templates:catcert:PL_expedient. Els altres dos casos fan servir la urn:iarxiu:2.0:templates:catcert:PL_document.
Aquestes plantilles han d'estar carregades a l'ens/fons a través de la Web de referència d'iArxiu (frontal web gestionat per l'arxiver). En cas contrari no es podrà ingressar cap paquet.

**mets.xml** per a paquets d'expedients:
```
<?xml version="1.0" encoding="UTF-8"?>
<mets:mets xmlns:mets="http://www.loc.gov/METS/" TYPE="urn:iarxiu:2.0:templates:catcert:PL_expedient">
	...
```
**mets.xml** per a paquets de documents:
```
<?xml version="1.0" encoding="UTF-8"?>
<mets:mets xmlns:mets="http://www.loc.gov/METS/" TYPE="urn:iarxiu:2.0:templates:catcert:PL_document">
	...
```

###2.2 Securització missatgeria
SoapUI permet signar les caçaleres SAML necessàries per poder comunicar-se amb iArxiu. El projecte ja incorpora una configuració (des de la Project View > Pestaña WS-Security Configurations) associada a un certificat CDA de prova.
Per a poder fer-ne ús, abans caldrà que es configuri a iArxiu a l'ens/fons corresponent els permisos necessaris per aquest certificat (en cas contrari iArxiu retornarà un error referent a que no es confia en aquesta TA, trusted application). 

![SoapUI Project View > WS-Security Configurations](/img/ws-security.PNG?raw=true "SoapUI Project View > WS-Security Configurations")

Cada petició aplica la política de securització (signa les capceleres SAML) automàticament:
![SoapUI Project View > WS-Security Configurations](/img/ws-security_request.PNG?raw=true "SoapUI Project View > WS-Security Configurations")

Aquesta configuració inicial cal sol·licitar-la a través de suport@aoc.cat.

## Casos d'ús
* **Expedient_UpDown_XAdES_Detached_Zip**
Paquet d'expedient amb 1 document i 1 signatura XAdES detached.
* **Doc_UpDown_PDF_300Kb_Zip**
Paquet d'expedient amb 1 document PDF signat (300kb).
* **Doc_UpDown_PDF_5MB_Zip**
Paquet d'expedient amb 1 document PDF signat (5MB).

Els 3 casos fan ús del mètode d'ingrés més adecuat: es pugen els documents de l'expedient (en un fitxer ZIP, mètode UploadOfflineZipIngest) a través d'un servlet http i amb el tiquet retornat es demana l'ingrés asíncron a iArxiu. Amb el segon identificador retornat al sol·licitar l'ingrés, es pot consultar l'estat d'aquest.
Un cop l'ingrés ha finalitzat (error o ok) els tests també sol·liciten la descàrrega del paquet (_no és necessari descarregar-lo quan s'ingressi a producció, només es fa amb finalitats de testing_).

``` Es recomana consultar l'estat de l'ingrés en cicles no inferiors als 15 segons per tal de no saturar el servei.```

## Estructura
* **PIT_expedient_XAdES_detached\\** (carpeta amb els fitxers per ingressar un expedient amb signatura XAdES detached i el mets.xml)
* **PIT_document_pdf\\** (carpeta amb els fitxers per ingressar un document PDF de 300Kb i el mets.xml)
* **PIT_document_5MB_pdf\\** (carpeta amb els fitxers per ingressar un document PDF de 5MB i el mets.xml)
* **download\\** (carpeta on es descarreguen els paquest ingressats en l'últim step)
* **PIT_expedient_XAdES_detached.zip** (zip del contingut de la carpeta PIT_expedient_XAdES_detached)
* **PIT_document_pdf.zip** (zip del contingut de la carpeta PIT_document_pdf)
* **PIT_document_5MB_pdf.zip** (zip del contingut de la carpeta PIT_document_5MB_pdf)
* **pre.properties** (properties per l'entorn de preproducció de proves)
* ***.xsd** (esquemes de la missatgeria)
* **ingest.wsdl** (descriptor per el ws d'ingrés)
* **dissemination.wsdl** (descriptor per el ws de consulta)
* **trustore\truststore-preproduccio.jks** (magatzem de certificats de confiança per a **preproducció**)
* **key\cda-1_vigent.pfx** (clau privada de proves per signar les peticions)
* **iarxiu-soapui-integracio-project.xml** (projecte SoapUI)

## Control d'unicitat
Quan s'ingressa un paquet iArxiu determina si aquest existeix ja o no al sistema fixant-se amb varis elements, entre ells certs camps de metadades del fitxer ```mets.xml```.
Modificant tan sols alguna d'aquestes metadades iArxiu ja no el detecta com duplicat i permetrà l'ingrés. Per poder reaprofitar els paquest d'exemple adjunts amb el projecte a través d'un petit scrit (```MetsModificator```) es llegeix el fitxer ```mets.xml```de cada paquet i se'n modifica alguna metadada.
Al primer step de cada TestCase (Update mets.xml) es crida a aquest script i es modifica alguna metadada sensible al control d'unicitat del ```mets.xml``` d'aquell paquet per poder tornar-se a ingressar automàticament, no cal configurar res.

