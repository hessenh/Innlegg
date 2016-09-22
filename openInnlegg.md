
I et tidligere [blogginnlegg](http://open.bekk.no/jakten-pa-fem-tusen-skatteberegninger-i-sekundet) snakket vi om hvordan skatteetaten er i ferd med å utvikle en ny plattform for forskudd- og skattebergninger, som vil erstatte dagens løsning på stormaskin. Målet med den nye plattform er at den har en arkitektur som er fleksibel, skalerbar, robust, og ikke minst fremtidsrettet. En indikator på at denne løsningen er fremtidsrettet, er hvis den enkelt kan flyttes over til en allmenn skyleverandør.

For å beregne skatten til alle skatteyterne i Norge kreves *skattegrunnlag* og *skatteplikt*, typisk formatert som XML-dokumenter. *Skattegrunnlag* inneholder verdiene av alt du eier, tjener, skylder og har av utgifter og fradrag, mens *skatteplikt* er relevant informasjon om deg, som alder, bosted og familieforhold.  

Stegene for skatteberegning er som følger:
1. Hent *skattegrunnlag* og *skatteplikt*
2. Kall skattebergnerfunksjon med de to dokumentene
3. Lagre resultat

Ved å kjøre skatteberegningsfunksjonen for alle skatteytere i Norge vil vi kunne finne hvor mye hver enkelt skal betale i skatt. I teorien vil ikke en slik omfattende beregning kjøres ofte, men i praksis er det interessant å kjøre den mange ganger i forbindelse med testing, analyser og rapportering. Det er dermed kritisk at en slik beregning er rask.

Løsningen er bygd med Amazon Web Services (AWS). AWS har lang fartstid og stiller med en rekke tjenester. Vår arkitektur tar i bruk følgende tjenester:

-   DynamoDB for datalagring
-   Lambda for å kalle beregningsfunksjonen
-   Kinesis som meldingstjeneste
-   Key Manangement Service (KMS) for nøkkelhåndtering og kryptering

# AWS-arkitektur
DynamoDB er en moderne NoSQL database som støtter et fleksibelt antall attributter, enkle og sammensatte primærnøkler og muligheten for å skrive og lese i grupper. DynamoDB har også støtte for triggere, som kaller en lambda-funksjon for endring i databasen. Lambda kjører et stykke kode når den mottar en hendelse, i vårt tilfelle, en skatteberegningsfunksjon.

Kinesis ble brukt som hendelsekilde for Lambda. Kinesis jobber med datastrømmer (*stream*), som kan deles opp i delstrømmer (*shards*). Dokumentasjonen sier at Lambda vil starte like mange samtidige eksekveringer som det er delstrømmer i Kinesis-strømmen, og dermed har vi en tilsynelatende enkel skaleringsmekanisme.

![Kinesis shards og lambda][kinesis-lambda]

Hendelsesforløpet er som følger:

1.  Klienten laster opp *skattegrunnlag* og *skatteplikt* til DynamoDB.
2.  Klienten mater Kinesis med fødselsnumre som identifikator og fordeler dem på delstrømmer.
3.  Lambda-funksjonen mottar hendelser fra Kinesis-strømmer og henter *skattegrunnlag* og *skatteplikt* fra DynamoDB, beregner skatt og skriver resultat til DynamoDB.

![Illustrasjon av AWS-arkitektur][aws-arkitektur]

# Konsept
For å gjøre det mulig for skatteetaten å bruke en serverløs løsing for skatteberegninger må dataene være tilstrekkelig sikret. Skattegrunnlaget og skatteplikten inneholder ikke sensitiv eller direkte identifiserende informasjon, men det er i teorien mulig å avlede informasjon som kan være kritisk for enkeltpersoner, for eksempel bostedet til mennesker som lever på hemmelig adresse. Vi bestemte oss for å kryptere skatteplikt og skattegrunnlag på klientsiden før de lastes opp til DynamoDB. Dette burde gjøre opplasting og lagring av dataene tilstrekkelig sikkert. Lambda-funksjonen henter dokumenter på samme måte som tidligere. Forskjellen er at for å kunne gjøre beregninger, må de nå dekrypteres. Når skattebergningen er fullført, krypteres resultatet før det sendes tilbake til databasen.

AWS har en tjeneste for nøkkelhåndtering kalt Key Management Service (KMS), som vi ville forsøke å bruke.  Man kan opprette nøkler, kalt Customer Master Key (CMK), og sette hvilke personer og tjenester som skal ha tilgang til dem. Nøklene slipper aldri ut av KMS, for å bruke nøklene i kryptografiske operasjoner må data sendes til KMS og bli kryptert der.

# Kryptering i KMS
For å komme i gang med KMS, implementerte vi noen enkle tester for å utforske funksjonaliteten den tilbød. Vi opprettet en CMK og brukte den til å kryptere *skattegrunnlag* og *skatteplikt*. Dette ble gjort ved å sende en *encryption request* til KMS med klarteksten lagt ved, for så å få tilbake en kryptert versjon.

```java
String keyId = "dummy-key";
ByteBuffer plaintext = ByteBuffer.wrap(new byte[]{1,2,3,4,5,6,7,8,9,0});
EncryptRequest req = new EncryptRequest();
req.withKeyId(keyId).withPlaintext(plaintext);
ByteBuffer ciphertext = kms.encrypt(req).getCiphertextBlob();
```

![krypterer med customer master key][server-side-kms]

I testen vår prøvde vi å laste opp 10 000 dokumenter til både *skattegrunnlag*- og *skatteplikt*-tabellene etter å ha kryptert dem. Det første vi oppdaget var at KMS ga feilmelding om at tjenesten bare godtar 100 forespørsler i sekundet. Et annet problem med denne løsningen var at for å kryptere et dokument måtte man sende det til KMS for så å få det krypterte dokumentet som svar. En siste begrensning er at datamengden er begrenset til 2 kB. Det vil i praksis ofte være for lite, og dermed uansett uaktelt. Den oppskriftsmessige tinærimngen er derfor å bruke *envelope encryption*.

# Envelope Encryption
Envelope Encryption (EE) går i korte trekk ut på følgende:

## Kryptering:
-   Bruk en unik datanøkkel for å kryptere dokumenter.
-   Krypter datanøkkelen med hovednøkkel (CMK)
-   Lagre det krypterte dokumentet sammen med den krypterte datanøkkelen.

## Dekryptering:
-   Hent dokumentet man skal dekryptere, sammen med den kryptere datanøkkelen
-   Bruk hovednøkkelen til å dekryptere datanøkkelen.
-   Dekrypter dokumentet med datanøkkelen.

På forespørsel om å generere en unik datanøkkel, vil KMS returnere en datanøkkel og en kryptert versjon av denne.

```java
// Generere ny datanøkkel
String keyId = "dummy-key";
AWSKMSClient kms = new AWSKMSClient();
GenerateDataKeyResult keyResult = kms.generateDataKey(
       new GenerateDataKeyRequest()
               .withKeySpec("AES_128")
               .withKeyId(keyId));
```

![hente datanøkkel fra kms][hent-datakey-kms]

Den ukrypterte datanøkkelen blir brukt til kryptering av dokumentet. Den krypterte datanøkkelen og det krypterte dokumentet lagres sammen i DynamoDB.

![kryptering og lagring][kryptering-og-lagring]

For å dekryptere dokumentet, henter man først den krypterte datanøkkelen sammen med dokumentet, og sender nøkkelen til KMS for dekryptering. KMS returnerer den dekrypterte nøkkelen, som brukes til å dekryptere dokumentet.

![dekryptere datanøkel med kms][dekryptere-datakey-kms]

Dette medførte at vi måtte ta hånd om kryptering/dekryptering selv. Vi bruker Java’s innebygde [Cipher](https://docs.oracle.com/javase/7/docs/api/javax/crypto/Cipher.html) bibliotek som er basert på Bouncy Castle for kryptering. Vi bruker AES-128, med GCM mode of operation uten padding. Cipher har ikke mulighet til å bruke AES-256, og selv om det hadde vært å foretrekke, er ikke dette noe stort tap fordi 128 bit regnes som tilstrekkelig sikkerhet. Vi bruker samme metode for å kryptere både for opplasting fra klient og i lambda-funksjonen.

```java
String keyId = "dummy-key";
byte [] dataKey = keyResult.getCiphertextBlob().array()
ByteBuffer plaintext = ByteBuffer.wrap(new byte[]{1,2,3,4,5,6,7,8,9,0});
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(dataKey, "AES"));
byte [] encryptedFile = cipher.doFinal(fil)
```

Vi bestemte oss for å bruke EE til å låse flere dokumenter med den samme datanøkkelen for å begrense antall kall til KMS. På den måten trengte vi bare å hente en ny datanøkkel en gang i blant. Dette valget ledet til en diskusjon om hvor mange dokumenter som kan krypteres med samme nøkkel. Vi prøvde oss i begynnelsen med ny nøkkel per 25. dokument, det viste seg å fremteles produsere for mange kall til KMS.
Vi innså at vi måtte ta høyde for at hver lambda-instans ville multiplisere antallet forespørsler til KMS både ved dekryptering og kryptering. For å redusere antallet forespørsler, implementerte vi caching av datanøkler ved dekryptering. Hver gang det ble sendt en forespørsel om å dekryptere en nøkkel ble både klartekst-nøkkelen og den krypterte nøkkelen lagret i en *map*. Vi skrudde også antallet nøkler videre ned til å kunne bruke samme nøkkel på 10 prosent av dokumentene. Prosessen gikk fremdeles for sent forid vi hadde fremdeles ikke tatt helt hensyn til hvor mange kall som ville bli generert med et høyt antall Kinesis-delstrømmer som trigget lambda. 10-prosent-grensen var heller ikke så godt gjennomtenkt, da den harde grensen på 100 kall i sekundet lett kunne overskrides dersom vi brukte mer enn 100 datanøkler. Etter å ha spurt mer erfarne kryptologer og google, bestemte vi oss for å gå for én nøkkel per tabell i DynamoDB. Det virket ikke som om dette ville gå hardt utover sikkerheten så lenge nøklene blir rullert ofte nok. I tillegg, for å spare tid når lambda-funksjonen skal kryptere beregnet skatt, gjenbrukte vi nøklene som ble cachet under dekryptering av *skatteplikt* og *skattegrunnlag*. Dette gjorde vi fordi vi ikke fant en enkel måte å begrense antallet datanøkler for kryptering av *beregnet skatt* til å være lavere enn antall delstrømmer. Det sparte oss også mange kall til til KMS.

![lambda-og-kms][lambda-og-kms]

Etter å ha gått over til en datanøkkel per tabell, endte vi opp med hastigheter som kunne måle seg med dem vi hadde sett før vi implementerte kryptering. Krypteringsoperasjonene gikk, som ventet, raskt fordi det er et ganske små dataamengder. Det som virkelig krevde tid var dataoverføring, mer spesifikt henting av datanøkler fra KMS. Ved å begrense oss til en nøkkel per tabell, begrenset vi også kall til KMS slik at det kun var den første dekrypteringsoperasjonen i hver lambda-funksjon som sendte en forespørsel om ny nøkkel. Etter det brukte den bare den nøkkelen den hadde fått sist. Når vi i tilleg gjenbrukte nøkler for kryptering av *beregnet skatt*, ble antallet kall veldig lavt.

# Sikkerhet
Det ER mulig å oppnå ett høyt nivå av sikkerhet for løsningen vår Ved å begrense nøkler og tilganger til “least privilege”. Kravene for dette er blant annet at nøklene kun skal kunne aksesseres fra nettet til klienten, skatteetaten i vår case, og internt i AWS løsningen. Bortset fra klienten skal kun lambda-instansene kunne lese og skrive til DynamoDB-tabellene. Tilgangen til nøklene må også ha *policies* som begrenser hvem som kan bruke dem og hva de kan brukes til. Dette gjelder hovedsaklig CMS-en. Nøklene må også begrenses slik at ingen menneskelig konto har tilgang til dem, kun lambda-prosessene. I tillegg må bruk av nøklene logges, slik at man kan sette opp alarmer som triggres hvis unormal oppførsel oppdages.

Alle disse kravene kan oppnås ved å bruke tjenester og funksjoner i AWS og AWS KMS. AWS Identity and Access Management (IAM) er en tjeneste som hjelper deg å sikre tilganger til de forskjellige resursene man har i AWS. Man kan legge inn brukere, grupper og roller som man kan legge rettigheter til. Brukerrettigheter var nødvendig for at vi skulle laste opp data til DynamoDB, og roller var nødvendig for å gi Lambda rettigheter mot Kinesis og DynamoDB. På samme måte kan man sette hvilke brukere eller roller som har tilgang til å bruke CMK i KMS.  

![key users][key-users]

CloudTrail er en tjeneste som logger API-kall mot AWS. Denne tjenesten leverer logger som gir informasjon hvem som utførte kall, tidspunkt, IP-adresse, hvilke parameter som var i forespørselen og hva AWS returnerte. CloudTrail lagrer loggene i en S3-database som er mulig å kryptere. Loggene kan også videresendes til CloudWatch, der man kan sette opp forskjellige alarmer.

# Kostnader
Som for de fleste tjenester fra AWS, så er KMS gratis i bruk opp til en grense. For KMS er grensen 20 000 forespørsler i måneden, deretter koster hver 10 000 forespørsler $0.03.

Hvis vi bruker 100 delstrømmer vil regnestykket over forespørsler bli følgende:

![kostnader-kms][kostnader-kms]
Prosessen fra å laste opp kryptert data til DynamoDB, kjøre beregninger på Lambda og laste ned data og dekryptere den vil innebære 204 kall mot KMS. Det vil si at vi kan kjøre den 98 ganger i månenden gratis, uavhengig av hvor mange skattytere vi beregner. Påfølgende kjøring vil enkeltvis koste $0,000612.


[kinesis-lambda]:https://bekkopen.blob.core.windows.net/attachments/1b4f1116-3702-4002-8f93-88d61d906993
[aws-arkitektur]:https://bekkopen.blob.core.windows.net/attachments/b869be92-e71a-4499-88ba-7412e51855d1
[server-side-kms]:https://bekkopen.blob.core.windows.net/attachments/6a147708-e045-4c49-a4a3-9b3eb06d5503
[hent-datakey-kms]:https://bekkopen.blob.core.windows.net/attachments/0e3f67b7-3559-45a3-be1c-a60b26b82e4c
[kryptering-og-lagring]:https://bekkopen.blob.core.windows.net/attachments/b59c84b6-75e5-4e3d-98c6-6dbf01a689c8
[dekryptere-datakey-kms]:https://bekkopen.blob.core.windows.net/attachments/a469bbe3-bb0e-4f8f-a4b8-d567b6388e58
[key-users]: https://bekkopen.blob.core.windows.net/attachments/56de4772-4a07-497c-a60c-4f333ea7a02d
[lambda-og-kms]:https://bekkopen.blob.core.windows.net/attachments/c9e9908f-910a-4635-8ff6-b9e6c6c41a6a
[kostnader-kms]:https://bekkopen.blob.core.windows.net/attachments/9fcb0cfc-10ab-4af7-a4d7-0f0003f0cf33
