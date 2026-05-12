# API CleverSuite — Documentazione di riferimento

CleverSuite è un CRM modulare. La sua API REST espone in lettura e scrittura le entità del sistema, suddivise in tre namespace:

| Namespace | Ambito |
|---|---|
| `/CleverBase` | Anagrafiche, ticketing e documenti (clienti, utenti, ticket, attività, allegati, ecc.) |
| `/CleverCRM` | Vendite, contratti, offerte, articoli, fatturazione, traffico telefonico |
| `/CleverPresence` | Rilevazione presenze, ferie, straordinari |

---

## Indice

1. [Base URL](#1-base-url)
2. [Autenticazione](#2-autenticazione)
3. [Convenzioni generali](#3-convenzioni-generali)
   - 3.1 [Operazioni e codici di stato](#31-operazioni-e-codici-di-stato)
   - 3.2 [Formato delle risposte](#32-formato-delle-risposte)
   - 3.3 [Filtri di ricerca](#33-filtri-di-ricerca)
   - 3.4 [Tipi e formati](#34-tipi-e-formati)
4. [Endpoint — CleverBase](#4-endpoint--cleverbase)
5. [Endpoint — CleverCRM](#5-endpoint--clevercrm)
6. [Endpoint — CleverPresence](#6-endpoint--cleverpresence)
7. [Esempi end-to-end](#7-esempi-end-to-end)

---

## 1. Base URL

Tutte le chiamate hanno la forma:

```
https://{host}{endpoint}
```

dove `{host}` è il dominio dell'installazione CleverSuite del cliente. Negli esempi seguenti useremo il placeholder `https://{host}`.

---

## 2. Autenticazione

Tutte le richieste devono includere due header HTTP:

| Header | Valore |
|---|---|
| `username` | Email dell'utente CleverSuite |
| `password` | Password dell'utente |

Esempio:

```bash
curl -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     https://{host}/CleverBase/Cliente/BA4A7337-6E46-474D-9DDD-44E76F5D09F5
```

---

## 3. Convenzioni generali

### 3.1 Operazioni e codici di stato

L'API segue uno schema CRUD uniforme per tutte le entità. I metodi supportati sono `GET`, `POST`, `PUT`, `DELETE`.

| Operazione | Metodo + URL | Body | Risposta successo | Risposta errore |
|---|---|---|---|---|
| Lettura singola risorsa | `GET /{endpoint}/{id}` | — | `200` + JSON dell'entità | `404` + plain text |
| Ricerca risorse | `GET /{endpoint}?Campo=valore&...` | — | `200` + JSON con array (vedi §3.2) | `400` + plain text |
| Inserimento | `POST /{endpoint}` | JSON dell'entità | `204` (senza contenuto) oppure `200` + ID della nuova risorsa in plain text | `500` + plain text |
| Aggiornamento | `PUT /{endpoint}` | JSON dell'entità con `ID` obbligatorio | `204` (senza contenuto) | `404` se ID non esiste, `500` + plain text |
| Eliminazione | `DELETE /{endpoint}/{id}` | — | `204` (senza contenuto) | `404` se ID non esiste, `500` + plain text |

Nelle risposte di errore (`400`, `404`, `500`) il body è in `text/plain` e contiene la descrizione del problema.

**Errore di autenticazione (`401 Unauthorized`).** Su qualsiasi operazione, se gli header `username`/`password` mancano o non corrispondono a un utente valido, l'API risponde con `401 Unauthorized` e una breve descrizione testuale.

**Aggiornamento parziale (`PUT`).** Nel body di una `PUT` solo il campo `ID` è obbligatorio: vengono aggiornati esclusivamente i campi effettivamente presenti nel body. I campi marcati come obbligatori per la creazione (colonna "Obbl." nelle tabelle che seguono), se inclusi nel body di una `PUT`, **non possono essere `null` né stringa vuota**: la richiesta verrebbe rifiutata. Per non modificarli basta ometterli dal body.

### 3.2 Formato delle risposte

**`GET /{endpoint}/{id}` — singola risorsa.** Restituisce direttamente l'oggetto JSON:

```json
{
    "ID": "5C404CC1-FD82-4B94-B6A1-D9D593376169",
    "Campo1": "Valore1",
    "Campo2": "Valore2",
    "Campo3": "Valore3"
}
```

**`GET /{endpoint}?...` — ricerca.** Restituisce un oggetto contenitore con un array di risorse. La chiave dell'array coincide con il nome dell'entità (per esempio `Cliente`, `Allegato`, `Attivita`):

```json
{
    "Risorsa": [
        { "ID": "...", "Campo1": "ValoreX",  ... },
        { "ID": "...", "Campo1": "ValoreY",  ... }
    ]
}
```

Se nessuna risorsa corrisponde ai filtri, l'array è omesso e il body è un oggetto vuoto:

```json
{}
```

**`POST /{endpoint}` — body di inserimento.** È un singolo oggetto JSON. Il campo `ID` è opzionale: se fornito dal client viene usato (deve essere un GUID), altrimenti il server genera l'ID e lo restituisce in plain text con stato `200`.

```json
{
    "Campo1": "Valore1",
    "Campo2": "Valore2",
    "Campo3": "Valore3"
}
```

**`PUT /{endpoint}` — body di aggiornamento.** È un singolo oggetto JSON, il campo `ID` è **obbligatorio**:

```json
{
    "ID": "5C404CC1-FD82-4B94-B6A1-D9D593376169",
    "Campo1": "Valore1",
    "Campo2": "Valore2",
    "Campo3": "Valore3"
}
```

### 3.3 Filtri di ricerca

Sulla `GET` di ricerca i parametri di query corrispondono ai nomi dei campi dell'entità.

- I campi che rappresentano un **identificativo** (`ID` della risorsa stessa o qualunque foreign key, riconoscibili dal prefisso `ID...`) fanno **match esatto**.
- Tutti gli altri campi (testo, codici, descrizioni) fanno **match parziale** (`LIKE`).
- Più parametri si combinano in **AND** logico.

Esempio — clienti con ragione sociale che contiene "Azienda" e città uguale a "Roma":

```
GET /CleverBase/Cliente?RagioneSociale=Azienda&Citta=Roma
```

### 3.4 Tipi e formati

| Tipo | Convenzione |
|---|---|
| **GUID** | Stringa nel formato canonico `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. Usato per `ID` e per tutte le foreign key. |
| **Booleano** | Numero intero: `−1` = vero, `0` = falso. |
| **Data** | Stringa nel formato `yyyy-MM-dd` (per esempio `2026-03-09`). |
| **Data e ora** | Stringa nel formato `yyyy-MM-dd HH:mm:ss` (per esempio `2026-03-09 16:23:57`). |
| **Ora** | Stringa nel formato `HH:mm:ss` (per esempio `09:00:00`). |
| **Allegati / file binari** | Stringa Base64 nel campo `ContentBytes` (o equivalente, vedi singola entità). Il campo `ContentType` riporta il MIME type. |
| **Stringa vuota** | I campi opzionali non valorizzati sono spesso restituiti come `""` (incluse le foreign key non impostate). |

---

## 4. Endpoint — CleverBase

### 4.1 Allegato

Allegati associati ai messaggi e ai ticket.

**Endpoint:** `/CleverBase/Allegato`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | sì | Identificativo della risorsa. |
| `IDMessaggio` | GUID | | FK verso [Messaggi](#49-messaggi). |
| `IDTicket` | GUID | | FK verso [Ticket](#417-ticket). |
| `Nome` | string | | Nome del file. |
| `ContentBytes` | string (Base64) | | Contenuto binario codificato in Base64. |
| `ContentType` | string | | MIME type del file (es. `application/pdf`). |
| `Dimensione` | integer | | Dimensione del file in byte. |

### 4.2 AllegatoAttivita

Allegati associati alle attività.

**Endpoint:** `/CleverBase/AllegatoAttivita`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDAttivita` | GUID | sì | FK verso [Attivita](#43-attivita). |
| `Nome` | string | sì | |
| `ContentBytes` | string (Base64) | sì | |
| `ContentType` | string | sì | |

### 4.3 Attivita

Attività (telefonate, email, interventi, note operative) associate a un utente, a un cliente/fornitore e a un riferimento (ticket, opportunità, ecc.).

**Endpoint:** `/CleverBase/Attivita`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | | FK verso [Utente](#420-utente). |
| `IDTipoAttivita` | GUID | sì | FK verso [TipoAttivita](#418-tipoattivita). |
| `IDCliente` | GUID | | FK verso [Utente](#420-utente). |
| `IDFornitore` | GUID | | FK verso [Utente](#420-utente). |
| `IDRiferimento` | GUID | | ID di un'altra entità a cui l'attività fa riferimento (Ticket, Opportunità, ecc.). |
| `IDUtenteCreato` | GUID | | FK verso [Utente](#420-utente). Utente che ha creato l'attività. |
| `DataAttivita` | datetime | | Data e ora a cui si riferisce l'attività. |
| `Durata` | integer | | Durata in minuti. |
| `DescrizioneDettagliata` | string | | |
| `DescrizioneBreve` | string | | |
| `Effettuato` | boolean | | `−1` = effettuata, `0` = da fare. |
| `InCalendario` | boolean | | |
| `TipoVisualizzazione` | enum | | Valori ammessi: `"Tutti"`, `"Solo io"`. |
| `DirittoChiamata` | enum (int) | | `0` = No, `1` = Remoto, `2` = Urbano, `3` = Extraurbano. |
| `OnSite` | boolean | | |
| `DataCreazione` | datetime | sì | |
| `CalcolaInOreLavorate` | boolean | | |
| `VisualizzaInConversazione` | boolean | | |
| `DescrizioneRiferimento` | string | | Descrizione testuale del riferimento (es. titolo del ticket). |

### 4.4 CategoriaTicket

Categorie di primo livello dei ticket.

**Endpoint:** `/CleverBase/CategoriaTicket`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Categoria` | string | sì | Nome della categoria. |

### 4.5 Cliente

Anagrafica cliente/fornitore.

**Endpoint:** `/CleverBase/Cliente`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDErpPrincipale` | string | | Codice gestionale principale. |
| `RagioneSociale` | string | sì | |
| `PartitaIVA` | string | | |
| `Telefono` | string | | |
| `CodiceFiscale` | string | | |
| `Indirizzo` | string | | |
| `Note` | string | | |
| `Email` | string | | |
| `PEC` | string | | |
| `CodiceSDI` | string | | Codice destinatario SDI (fatturazione elettronica). |
| `Citta` | string | | |
| `Provincia` | string | | Sigla provincia. |
| `CAP` | string | | |
| `Tipo` | enum (int) | | `10` = Cliente Potenziale, `20` = Cliente, `30` = Fornitore. |
| `Fax` | string | | |
| `IBAN` | string | | |
| `ConsensoPrivacy` | boolean | | |
| `ConsensoInvioMail` | boolean | | |
| `NomeBanca` | string | | |
| `Provenienza` | string | | |
| `TipoContribuente` | enum | | `"Persona Giuridica"`, `"Persona Fisica"`, `"Pubblica Amministrazione"`. |
| `Premium` | boolean | | |
| `IDCondizionePagamento` | GUID | | FK verso [CondizionePagamento](#47-condizionepagamento). |
| `FornitoreNumerazioni` | boolean | | |
| `Insolvente` | boolean | | |
| `FornitoreOrdini` | boolean | | |
| `LegaleRappresentante` | string | | |
| `CodFiscLegaleRappresentante` | string | | |
| `IDErpAlternativo` | string | | |
| `Agenzia` | boolean | | |
| `RiferimentoAgente` | string | | |
| `FornitoreStrategico` | boolean | | |
| `BloccoAmministrativo` | boolean | | |

### 4.6 Comune

Anagrafica comuni italiani.

**Endpoint:** `/CleverBase/Comune`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | sì | |
| `Nome` | string | | |
| `Provincia` | string | | Sigla provincia. |
| `Regione` | string | | |
| `CodiceISTAT` | integer | | |
| `CAP` | string | | |

### 4.7 CondizionePagamento

Anagrafica delle condizioni di pagamento applicabili a contratti, offerte, fatture.

**Endpoint:** `/CleverBase/CondizionePagamento`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `IDERPPrincipale` | string | sì | Codice gestionale principale. |
| `FineMese` | boolean | | |
| `Giorni` | integer | sì | Giorni di scadenza. |
| `IDERPAlternativo` | string | | |
| `MetodoDiPagamento` | enum | | `"Contanti"`, `"Assegno"`, `"Bonifico"`, `"RIBA"`, `"SEPA Core"`, `"SEPA B2B"`. |
| `CostoIncasso` | number | | |

### 4.8 Documento

Documenti caricati e collegati a un cliente (offerte, ordini, contratti scansionati, ecc.).

**Endpoint:** `/CleverBase/Documento`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtenteStaff` | GUID | | FK verso [Utente](#420-utente). |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `IDTipologiaDocumento` | GUID | | FK verso [TipologiaDocumento](#419-tipologiadocumento). |
| `IDRiferimento` | GUID | | |
| `DescrizioneRiferimento` | string | | |
| `DataDocumento` | date | | |
| `DataCaricamento` | date | sì | |
| `Descrizione` | string | sì | |
| `NomeFile` | string | | |
| `ContentType` | string | | |
| `ContentBytes` | string (Base64) | | |
| `Dimensione` | integer | | |
| `Note` | string | | |
| `Firma` | string | | |
| `Firmatario` | string | | |
| `DataFirma` | date | | |
| `InviatoACliente` | boolean | | |
| `DataUltimoInvio` | date | | |
| `InviatoA` | string | | |
| `Vistato` | boolean | | |
| `RagioneSociale` | string | | Ragione sociale del cliente al momento del caricamento. |

### 4.9 Messaggi

Email e messaggi della conversazione del CRM.

**Endpoint:** `/CleverBase/Messaggio`

> Nelle risposte di ricerca l'array è esposto sotto la chiave `Messaggio` (singolare).

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | sì | FK verso [Utente](#420-utente). |
| `IDRiferimento` | GUID | | |
| `Corpo` | string | | Corpo del messaggio (può contenere HTML). |
| `DataMessaggio` | datetime | sì | |
| `Oggetto` | string | sì | |
| `Inviato` | boolean | | |
| `InoltratoA` | string | | |
| `Tipo` | string | | Es. `"Risposta"`. |
| `CC` | string | | Destinatari in copia conoscenza. |

### 4.10 MessaggioWhatsapp

Messaggi WhatsApp ricevuti/inviati associati a un contatto.

**Endpoint:** `/CleverBase/MessaggioWhatsapp`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | string | | Identificativo WhatsApp del messaggio (formato `wamid.*`). |
| `IDUtenteStaff` | GUID | | FK verso [Utente](#420-utente). |
| `IDNumerazione` | string | sì | Numerazione interna che ha gestito il messaggio. |
| `NumeroContatto` | string | sì | Numero di telefono del contatto esterno. |
| `DataMessaggio` | datetime | sì | |
| `Gestito` | boolean | | |
| `Corpo` | string | | |
| `Allegato` | string (Base64) | | Contenuto binario dell'allegato. |
| `Inoltrato` | string | | |
| `MimeTypeAllegato` | string | | |
| `NomeAllegato` | string | | |
| `IDGestitoDa` | GUID | | FK verso [Utente](#420-utente). |
| `IDRiferimento` | GUID | | |
| `DescrizioneRiferimento` | string | | |
| `NicknameContatto` | string | | |
| `Template` | boolean | | |

### 4.11 RapportoRisolutivo

Rapportini di chiusura di un ticket.

**Endpoint:** `/CleverBase/RapportoRisolutivo`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDTicket` | GUID | | FK verso [Ticket](#417-ticket). |
| `IDSottocategoriaTicket` | GUID | sì | FK verso [SottocategoriaTicket](#416-sottocategoriaticket). |
| `Titolo` | string | sì | |
| `NoteCliente` | string | sì | Note visibili al cliente. |
| `IDUtente` | GUID | | FK verso [Utente](#420-utente). |
| `NoteTecniche` | string | | Note interne. |
| `OreLavorateOnSite` | number | | |
| `OreLavorateDaRemoto` | number | | |
| `DirittoDiChiamata` | enum (int) | | `0` = No, `1` = Remoto, `2` = Urbano, `3` = Extraurbano. |
| `Fatturato` | boolean | | |
| `InAssistenza` | boolean | | |
| `IDFattura` | GUID | | FK verso [Fatture](#514-fatture). |
| `AssistenzaSpecialistica` | boolean | | |

### 4.12 Reparto

Reparti dell'azienda che gestiscono i ticket.

**Endpoint:** `/CleverBase/Reparto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Reparto` | string | sì | |
| `AbilitaRapportoRisolutivo` | boolean | | |
| `SollecitoAutomatico` | boolean | | |
| `RicevutaAperturaTicket` | boolean | | |
| `InviaRapportoRisolutivo` | boolean | | |
| `Firma` | string | | |
| `IncludiRiferimentoInApertura` | boolean | | |
| `RichiediOrePerChiusura` | boolean | | |
| `MostraInHelpdeskManager` | boolean | | |
| `MostraInPortaleClienti` | boolean | | |

### 4.13 RichiestaAnagrafica

Richieste di creazione/modifica anagrafica cliente in attesa di approvazione.

**Endpoint:** `/CleverBase/RichiestaAnagrafica`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `RagioneSociale` | string | sì | |
| `PartitaIVA` | string | | |
| `CodiceFiscale` | string | | |
| `TelefonoPrincipale` | string | | |
| `Indirizzo` | string | | |
| `Citta` | string | | |
| `CAP` | string | | |
| `Provincia` | string | | |
| `Email` | string | | |
| `PEC` | string | | |
| `CodiceSDI` | string | | |
| `Tipo` | string | | Es. `"Modifica"`, `"Nuova"`. |
| `Fax` | string | | |
| `IBAN` | string | | |
| `ConsensoPrivacy` | boolean | | |
| `ConsensoInvioMail` | boolean | | |
| `NomeBanca` | string | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `IDComuneSedeLegale` | GUID | | FK verso [Comune](#46-comune). |
| `Confermata` | boolean | | |
| `Processata` | boolean | | |
| `DataRichiesta` | date | | |
| `TipoAnagrafica` | enum | sì | `"Persona Fisica"`, `"Persona Giuridica"`, `"Pubblica Amministrazione"`. |
| `IDCondizionePagamento` | GUID | | FK verso [CondizionePagamento](#47-condizionepagamento). |

### 4.14 SedeCliente

Sedi operative/secondarie associate a un cliente.

**Endpoint:** `/CleverBase/SedeCliente`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `RagioneSociale` | string | | |
| `Telefono` | string | | |
| `Indirizzo` | string | | |
| `Citta` | string | | |
| `CAP` | string | | |
| `Provincia` | string | | |
| `Note` | string | | |
| `Email` | string | | |
| `Codice` | string | sì | Codice progressivo della sede. |
| `Attiva` | boolean | | |

### 4.15 SedeRichiestaAnagrafica

Sedi associate a una richiesta di anagrafica.

**Endpoint:** `/CleverBase/SedeRichiestaAnagrafica`

> Nelle risposte di ricerca l'array è esposto sotto la chiave `SedeCliente`.

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `RagioneSociale` | string | | |
| `Telefono` | string | | |
| `Indirizzo` | string | | |
| `Citta` | string | | |
| `CAP` | string | | |
| `Provincia` | string | | |
| `IDComune` | GUID | | FK verso [Comune](#46-comune). |
| `Note` | string | | |
| `Email` | string | | |
| `Codice` | string | sì | |
| `Attiva` | boolean | | |
| `IDRichiestaAnagrafica` | GUID | sì | FK verso [RichiestaAnagrafica](#413-richiestaanagrafica). |

### 4.16 SottocategoriaTicket

Sottocategorie di una categoria ticket.

**Endpoint:** `/CleverBase/SottocategoriaTicket`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCategoriaTicket` | GUID | sì | FK verso [CategoriaTicket](#44-categoriaticket). |
| `Sottocategoria` | string | sì | |

### 4.17 Ticket

Richieste di assistenza dei clienti.

**Endpoint:** `/CleverBase/Ticket`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `IDReparto` | GUID | sì | FK verso [Reparto](#412-reparto). |
| `IDUtenteCliente` | GUID | | FK verso [Utente](#420-utente). |
| `IDUtenteStaff` | GUID | | FK verso [Utente](#420-utente). |
| `IDUtenteUltimaModifica` | GUID | | FK verso [Utente](#420-utente). |
| `Oggetto` | string | sì | |
| `DataApertura` | datetime | sì | |
| `DataChiusura` | datetime | | |
| `RiferimentoTicket` | string | sì | Codice univoco del ticket (es. `TCK-227242`). |
| `DataUltimaModifica` | datetime | sì | |
| `Note` | string | | |
| `StatoTicket` | enum | | `"Aperto"`, `"In Lavorazione"`, `"Chiuso"`, `"Sospeso"`. |
| `Priorita` | enum | | `"Bassa"`, `"Media"`, `"Alta"`. |
| `IDCategoria` | GUID | | FK verso [SottocategoriaTicket](#416-sottocategoriaticket). |

### 4.18 TipoAttivita

Tipi di attività configurabili (segreteria, intervento on-site, telefonata, ecc.).

**Endpoint:** `/CleverBase/TipoAttivita`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `Colore` | string | | Colore in formato numerico (compatibile col CRM). |
| `InCalendario` | boolean | | |
| `Ambito` | enum | | `"Ticket"`, `"Canoni"`, `"Merci"`, `"Commerciale"`, `"Presenze"`, `"Veicoli"`, `"Ordini"`. |
| `TestoAvvisoEmail` | string | | |
| `AvanzaOpportunitaA` | string | | |
| `TastoRapido` | boolean | | |
| `OreLavorateDefault` | number | | |
| `VisualizzaInConversazione` | boolean | | |
| `CalcolaInOreLavorate` | boolean | | |
| `PermettiGestitoDettaglioOrdine` | boolean | | |
| `Bloccato` | boolean | | |
| `ColoreRGB` | string | | Colore in formato `R, G, B`. |

### 4.19 TipologiaDocumento

Tipologie configurabili di documenti.

**Endpoint:** `/CleverBase/TipologiaDocumento`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Tipo` | string | sì | |

### 4.20 Utente

Utenti del sistema (staff e contatti dei clienti).

**Endpoint:** `/CleverBase/Utente`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). Per gli utenti contatto. |
| `IDProfilo` | GUID | | FK verso il profilo di permessi. |
| `Email` | string | sì | |
| `Password` | string | | Password (hash). |
| `Nome` | string | | |
| `Cognome` | string | | |
| `Validita` | date | | Data di scadenza dell'utenza. |
| `Ruolo` | enum (int) | | `3` = Contatto, `100` = Staff, `1000` = Amministratore Sistema, `10000` = Anonimo. |
| `Telefono` | string | | |
| `Cellulare` | string | | |
| `InformazioniAggiuntive` | string | | |
| `AlertAssegnazioneTicket` | boolean | | |
| `UsernamePBX` | string | | |
| `PasswordPBX` | string | | |
| `UsernameERP` | string | | |
| `PasswordERP` | string | | |
| `UltimaVersioneVisualizzata` | string | | |
| `AvvisoGiornaliero` | string | | |
| `ResocontoGiornaliero` | boolean | | |
| `Agente` | boolean | | |
| `MFA` | boolean | | |
| `SecretMFA` | string | | |
| `InHelpdeskManager` | boolean | | |

---

## 5. Endpoint — CleverCRM

### 5.1 Articolo

Catalogo degli articoli (merci e canoni di servizio).

**Endpoint:** `/CleverCRM/Articolo`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Tipo` | string | sì | Tipologia dell'articolo (es. `"Merce"`, `"Canone"`). |
| `Codice` | string | sì | Codice prodotto univoco. |
| `DescrizioneBreve` | string | sì | |
| `Prezzo` | number | sì | |
| `PrezzoAttivazione` | number | | |
| `IDCategoriaMerceologica` | GUID | | FK verso [CategoriaMerceologica](#55-categoriamerceologica). |
| `IDCategoriaOmogenea` | GUID | | FK verso [CategoriaOmogenea](#54-categoriaomogenea). |
| `Abilitato` | boolean | | |
| `CliAssociato` | boolean | | |
| `UdM` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `IDMarcaArticolo` | GUID | | FK verso [MarcaArticolo](#518-marcaarticolo). |
| `Servizio` | boolean | | |
| `RicorrenzaPredefinita` | enum | | `"Mensile"`, `"Bimestrale"`, `"Trimestrale"`, `"Quadrimestrale"`, `"Semestrale"`, `"Annuale"`. |
| `AssistenzaCompresa` | boolean | | |
| `DescrizioneOfferta` | string | | |

### 5.2 CanoneOfferta

Righe canone di un'offerta.

**Endpoint:** `/CleverCRM/CanoneOfferta`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDOfferta` | GUID | sì | FK verso [Offerta](#523-offerta). |
| `IDArticolo` | GUID | | FK verso [Articolo](#51-articolo). |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `IDDettaglioContratto` | GUID | | FK verso [DettagliContratti](#510-dettaglicontratti). |
| `Posizione` | integer | | Posizione nella riga d'offerta. |
| `TipologiaArticolo` | string | | Sempre `"Canone"`. |
| `CodiceProdotto` | string | | |
| `Descrizione` | string | | |
| `Note` | string | | |
| `UnitaDiMisura` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `Quantita` | number | | |
| `Prezzo` | number | | |
| `CostoDiAttivazione` | number | | |
| `DescrizioneDettagliata` | string | | |
| `PercorsoImmagine` | string | | |
| `Sconto` | number | | |
| `ScontoAttivazione` | number | | |
| `StatoServizio` | string | | |
| `TipoSconto` | enum | | `"Percentuale"`, `"Prezzo"`. |
| `TipoScontoAttivazione` | enum | | `"Percentuale"`, `"Prezzo"`. |
| `RicorrenzaPagamento` | enum | | `"Mensile"`, `"Bimestrale"`, `"Trimestrale"`, `"Quadrimestrale"`, `"Semestrale"`, `"Annuale"`. |
| `GiàAttivo` | boolean | | |
| `DurataFatturazione` | integer | | Durata in mesi. |
| `Causale` | enum | | `"Attivazione"`, `"Variazione"`, `"Cessazione"`. |
| `Immagine` | string (Base64) | | |
| `Totale` | number | | Totale calcolato (scontato). |
| `TotaleAttivazione` | number | | |
| `TotaleNonScontato` | number | | |
| `TotaleNonScontatoAttivazioni` | number | | |

### 5.3 CanoneOrdine

Righe canone di un ordine.

**Endpoint:** `/CleverCRM/CanoneOrdine`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDOrdine` | GUID | sì | FK verso [Ordine](#525-ordine). |
| `IDArticolo` | GUID | sì | FK verso [Articolo](#51-articolo). |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `IDDettaglioOfferta` | GUID | sì | FK verso [CanoneOfferta](#52-canoneofferta) / [MerceOfferta](#519-merceofferta). |
| `IDDettaglioContratto` | GUID | | FK verso [DettagliContratti](#510-dettaglicontratti). |
| `Causale` | enum | | `"Attivazione"`, `"Variazione"`, `"Cessazione"`. |
| `TipologiaArticolo` | string | | Sempre `"Canone"`. |
| `Descrizione` | string | | |
| `Quantita` | number | | |
| `NumeroPratica` | string | | |
| `Gestito` | boolean | | |

### 5.4 CategoriaOmogenea

Categorie omogenee per la classificazione degli articoli.

**Endpoint:** `/CleverCRM/CategoriaOmogenea`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |

### 5.5 CategoriaMerceologica

Categorie merceologiche.

**Endpoint:** `/CleverCRM/CategoriaMerceologica`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |

### 5.6 CausaleTrasporto

Causali da utilizzare nei documenti di trasporto.

**Endpoint:** `/CleverCRM/CausaleTrasporto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `Giacenza` | enum | sì | `"Aumenta"`, `"Decrementa"`, `"Ignora"`. |
| `Disponibilita` | enum | sì | `"Aumenta"`, `"Decrementa"`, `"Ignora"`. |
| `SegueFattura` | boolean | | |
| `DaControllare` | boolean | | |

### 5.7 ChiamataCliente

Traffico telefonico (CDR) associato ai clienti.

**Endpoint:** `/CleverCRM/ChiamataCliente`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `Sorgente` | string | sì | Numero chiamante. |
| `Destinazione` | string | sì | Numero chiamato. |
| `DurataInSecondi` | integer | sì | |
| `Data` | datetime | sì | |
| `Fatturato` | boolean | | |
| `CostoChiamata` | number | | |
| `Note` | string | | |
| `Tipo` | enum | sì | `"cps"`, `"wlr"`, `"voip"`, `"800"`, `"telegramma"`. |
| `ImportFileName` | string | sì | Nome del file CDR di origine. |
| `FornitoreChiamate` | enum | sì | `"twt"`, `"clouditalia"`. |
| `IDNumerazioneContratto` | GUID | | FK verso [NumerazioneContratto](#521-numerazionecontratto). |
| `IDFattura` | GUID | | FK verso [Fatture](#514-fatture). |
| `TipoNumeroDestinazione` | enum | | `"Nazionale"`, `"Internazionale"`, `"Cellulare"`, `"800"`, `"Pagamento"`. |
| `Direttrice` | string | | |

### 5.8 CLINumerazione

CLI (Calling Line Identification) associati a una numerazione di contratto.

**Endpoint:** `/CleverCRM/CLINumerazione`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDNumerazioneContratto` | GUID | sì | FK verso [NumerazioneContratto](#521-numerazionecontratto). |
| `Numerazione` | string | sì | |

### 5.9 Contratto

Contratti attivi con i clienti.

**Endpoint:** `/CleverCRM/Contratto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `IDCondizionePagamento` | GUID | sì | FK verso [CondizionePagamento](#47-condizionepagamento). |
| `Numero` | string | sì | Numero del contratto (es. `2026/0024`). |
| `Data` | datetime | sì | |
| `Descrizione` | string | sì | |
| `Banca` | string | | |
| `IBAN` | string | | |
| `Note` | string | | |
| `SDI` | string | | |
| `Disdettato` | boolean | | |
| `DataDisdetta` | datetime | | |
| `CIG` | string | | |
| `CUP` | string | | |
| `BonusTelefonici` | enum | | `"No"`, `"1000 Minuti"`, `"2000 Minuti"`, `"3000 Minuti"`, `"4000 Minuti"`, `"5000 Minuti"`, `"Flat"`. |

### 5.10 DettagliContratti

Righe di dettaglio di un contratto.

**Endpoint:** `/CleverCRM/DettaglioContratto`

> Nelle risposte di ricerca l'array è esposto sotto la chiave `DettaglioContratto`.

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDArticolo` | GUID | sì | FK verso [Articolo](#51-articolo). |
| `IDContratto` | GUID | sì | FK verso [Contratto](#59-contratto). |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `Posizione` | integer | | |
| `Descrizione` | string | | |
| `DescrizioneDettagliata` | string | | |
| `Note` | string | | |
| `RicorrenzaPagamento` | enum | | `"Mensile"`, `"Bimestrale"`, `"Trimestrale"`, `"Quadrimestrale"`, `"Semestrale"`, `"Annuale"`. |
| `UnitaDiMisura` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `Quantita` | number | sì | |
| `Prezzo` | number | sì | |
| `Sconto` | number | | |
| `CostoAttivazione` | number | | |
| `ScontoAttivazione` | number | | |
| `Attivo` | boolean | | |
| `DataAttivazione` | date | | |
| `DataInizioFatturazione` | date | | |
| `DataFineFatturazione` | date | | |
| `Cessato` | boolean | | |
| `NumeroPratica` | string | | |
| `Fatturabile` | boolean | | |
| `DataAccettazione` | datetime | | |
| `DurataFatturazione` | integer | | |
| `UltimaAttivitaCommerciale` | enum | | `"Attivazione"`, `"Cessazione"`, `"Variazione"`. |
| `AttivazioneFatturata` | boolean | | |

### 5.11 DettaglioDocumentoDiTrasporto

Righe di dettaglio di un documento di trasporto.

**Endpoint:** `/CleverCRM/DettaglioDocumentoDiTrasporto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDArticolo` | GUID | | FK verso [Articolo](#51-articolo). |
| `IDCausaleTrasporto` | GUID | sì | FK verso [CausaleTrasporto](#56-causaletrasporto). |
| `CodiceProdotto` | string | sì | |
| `Descrizione` | string | | |
| `UnitaDiMisura` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `Quantita` | number | | |
| `IDDocumentoDiTrasporto` | GUID | sì | FK verso [DocumentoDiTrasporto](#513-documentoditrasporto). |
| `PrezzoUnitario` | number | sì | |
| `IDRiferimento` | GUID | | |
| `TipoRiferimento` | enum | | `"Contratto"`, `"Ticket"`, `"Ordine"`. |
| `ScontoUnitario` | number | | |
| `TipoSconto` | enum | | `"Percentuale"`, `"Prezzo"`. |
| `Rientrato` | boolean | | |

### 5.12 DettaglioFattura

Righe di dettaglio di una fattura.

**Endpoint:** `/CleverCRM/DettaglioFattura`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDFattura` | GUID | sì | FK verso [Fatture](#514-fatture). |
| `IDRiferimento` | GUID | | |
| `TipoRiferimento` | string | | Es. `"Canone Contratto"`. |
| `Posizione` | integer | | |
| `TipoRiga` | enum | sì | `"Merce"`, `"Canone"`, `"Servizio"`, `"Nota"`. |
| `Descrizione` | string | | |
| `UnitaDiMisura` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `Quantita` | number | sì | |
| `Prezzo` | number | sì | |
| `IDArticolo` | GUID | | FK verso [Articolo](#51-articolo). |
| `DataInizioCompetenza` | date | | |
| `DataFineCompetenza` | date | | |
| `Sconto` | number | | |
| `InFatturaElettronica` | boolean | | |

### 5.13 DocumentoDiTrasporto

Documenti di trasporto (DDT).

**Endpoint:** `/CleverCRM/DocumentoDiTrasporto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `IDUtente` | GUID | | FK verso [Utente](#420-utente). |
| `IDGestionale` | string | | Codice del documento nel gestionale esterno. |
| `Data` | datetime | | |
| `Numero` | string | sì | |
| `Descrizione` | string | sì | |
| `RagioneSociale` | string | | |
| `Citta` | string | | |
| `Indirizzo` | string | | |
| `Provincia` | string | | |
| `CAP` | string | | |
| `Email` | string | | |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `Vettore` | string | | |
| `DataPartenza` | datetime | sì | |
| `AspettoBeni` | enum | | `"Scatole"`, `"Bobine"`, `"Sfuso"`, `"A Vista"`. |
| `ResponsabileTrasporto` | enum | | `"Mittente"`, `"Destinatario"`. |
| `NumeroColli` | integer | | |
| `Peso` | number | | |
| `Note` | string | | |
| `Annullato` | boolean | | |
| `CIG` | string | | |
| `CUP` | string | | |
| `Fatturato` | boolean | | |
| `IDFattura` | GUID | | FK verso [Fatture](#514-fatture). |
| `DettagliInvio` | string | | |

### 5.14 Fatture

Testate delle fatture emesse.

**Endpoint:** `/CleverCRM/Fattura`

> Nelle risposte di ricerca l'array è esposto sotto la chiave `Fattura`.

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `IDERP` | integer | | Identificativo nel gestionale esterno. |
| `Numero` | string | | |
| `Data` | date | sì | |
| `CIG` | string | | |
| `CUP` | string | | |
| `ModelloContabile` | enum | | `"Fattura"`, `"Fattura Differita"`. |
| `IDCondizionePagamento` | GUID | | FK verso [CondizionePagamento](#47-condizionepagamento). |
| `Tag` | string | | |
| `Riferimento` | string | | Riferimento esterno (es. numero ordine). |
| `DataRiferimento` | date | | |

### 5.15 GruppoPrefissi

Gruppi di prefissi telefonici.

**Endpoint:** `/CleverCRM/GruppoPrefissi`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `Note` | string | | |

### 5.16 ImmagineArticolo

Immagini associate a un articolo.

**Endpoint:** `/CleverCRM/ImmagineArticolo`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDArticolo` | GUID | sì | FK verso [Articolo](#51-articolo). |
| `Tipo` | string | sì | Sempre `"Immagine"`. |
| `ContentBytes` | string (Base64) | sì | Contenuto binario dell'immagine/PDF. |
| `ContentType` | string | sì | MIME type. |
| `Nome` | string | sì | |

### 5.17 ListinoChiamate

Listini di tariffazione delle chiamate.

**Endpoint:** `/CleverCRM/ListinoChiamate`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `Note` | string | | |
| `Direttrice` | string | | |

### 5.18 MarcaArticolo

Marche associate agli articoli.

**Endpoint:** `/CleverCRM/MarcaArticolo`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Nome` | string | sì | |

### 5.19 MerceOfferta

Righe merce di un'offerta.

**Endpoint:** `/CleverCRM/MerceOfferta`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDOfferta` | GUID | sì | FK verso [Offerta](#523-offerta). |
| `IDArticolo` | GUID | | FK verso [Articolo](#51-articolo). |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `Posizione` | integer | | |
| `TipologiaArticolo` | string | | Sempre `"Merce"`. |
| `CodiceProdotto` | string | | |
| `Descrizione` | string | | |
| `Note` | string | | |
| `UnitaDiMisura` | enum | | `"NR"`, `"H"`, `"PZ"`. |
| `Quantita` | number | | |
| `Prezzo` | number | | |
| `DescrizioneDettagliata` | string | | |
| `PercorsoImmagine` | string | | |
| `Sconto` | number | | |
| `StatoServizio` | string | | |
| `TipoSconto` | enum | | `"Percentuale"`, `"Prezzo"`. |
| `Causale` | enum | | `"Vendita"`, `"Ritiro"`, `"Comodato"`. |
| `Immagine` | string (Base64) | | |
| `Totale` | number | | |
| `TotaleNonScontato` | number | | |

### 5.20 MerceOrdine

Righe merce di un ordine.

**Endpoint:** `/CleverCRM/MerceOrdine`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDOrdine` | GUID | sì | FK verso [Ordine](#525-ordine). |
| `IDArticolo` | GUID | sì | FK verso [Articolo](#51-articolo). |
| `IDSedeCliente` | GUID | | FK verso [SedeCliente](#414-sedecliente). |
| `IDDettaglioOfferta` | GUID | sì | FK verso [MerceOfferta](#519-merceofferta). |
| `Causale` | enum | | `"Vendita"`, `"Ritiro"`, `"Comodato"`. |
| `TipologiaArticolo` | string | | Sempre `"Merce"`. |
| `Descrizione` | string | | |
| `Quantita` | number | | |
| `NumeroPratica` | string | | |
| `Gestito` | boolean | | |

### 5.21 NumerazioneContratto

Numerazioni telefoniche associate a un contratto.

**Endpoint:** `/CleverCRM/NumerazioneContratto`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDFornitore` | GUID | | FK verso [Cliente](#45-cliente) (cliente di tipo Fornitore). |
| `IDCanoneCollegato` | GUID | sì | FK verso [DettagliContratti](#510-dettaglicontratti) (riga canone del contratto). |
| `Tipo` | enum | | `"VoIP"`, `"PSTN"`, `"ISDN"`, `"PRI"`. |
| `GNR` | enum | | `"No"`, `"10"`, `"100"`, `"1000"`. |
| `Suffisso` | string | | Suffisso del numero. |
| `InPortabilita` | boolean | | |
| `CodiceDiMigrazione` | string | | |
| `IDListinoChiamate` | GUID | | FK verso [ListinoChiamate](#517-listinochiamate). |
| `IDPrefissoTelefonico` | GUID | | FK verso [PrefissoTelefonico](#527-prefissotelefonico). |
| `Bonus` | enum | | `"No"`, `"1000 Minuti"`, `"2000 Minuti"`, `"3000 Minuti"`, `"4000 Minuti"`, `"5000 Minuti"`, `"Flat"`. |
| `CostoGestionePratica` | number | | |
| `CodiceDiMigrazioneOperatorePrecedente` | string | | |

### 5.22 NumerazioneOfferta

Numerazioni telefoniche presenti su un'offerta.

**Endpoint:** `/CleverCRM/NumerazioneOfferta`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDFornitore` | GUID | | FK verso [Cliente](#45-cliente) (cliente di tipo Fornitore). |
| `IDCanoneCollegato` | GUID | | FK verso [CanoneOfferta](#52-canoneofferta). |
| `Tipo` | enum | | `"VoIP"`, `"PSTN"`, `"ISDN"`, `"PRI"`. |
| `GNR` | enum | | `"No"`, `"10"`, `"100"`, `"1000"`. |
| `Suffisso` | string | | |
| `InPortabilita` | boolean | | |
| `CodiceDiMigrazione` | string | | |
| `CodiceDiMigrazioneOperatorePrecedente` | string | | |
| `IDListinoChiamate` | GUID | | FK verso [ListinoChiamate](#517-listinochiamate). |
| `IDPrefissoTelefonico` | GUID | | FK verso [PrefissoTelefonico](#527-prefissotelefonico). |
| `OperatorePrecedente` | string | | |
| `TecnologiaOperatorePrecedente` | string | | |
| `CostoGestionePratica` | number | | |

### 5.23 Offerta

Offerte commerciali emesse verso un cliente.

**Endpoint:** `/CleverCRM/Offerta`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | sì | FK verso [Utente](#420-utente). |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `IDCondizionePagamento` | GUID | | FK verso [CondizionePagamento](#47-condizionepagamento). |
| `IDOpportunita` | GUID | | FK verso [Opportunita](#524-opportunita). |
| `Numero` | string | | |
| `Data` | datetime | | |
| `Descrizione` | string | sì | |
| `RagioneSociale` | string | | |
| `Indirizzo` | string | | |
| `CAP` | string | | |
| `Citta` | string | | |
| `Provincia` | string | | |
| `Banca` | string | | |
| `IBAN` | string | | |
| `Note` | string | | |
| `Email` | string | | |
| `SDI` | string | | |
| `PEC` | string | | |
| `PartitaIVA` | string | | |
| `CodiceFiscale` | string | | |
| `Status` | enum | | `"Attesa Conferma"`, `"Accettato"`, `"Rifiutato"`, `"Scaduto"`, `"Attesa Contratto"`. |
| `Telefono` | string | | |
| `Fax` | string | | |
| `DataAccettazione` | datetime | | |
| `UrlFirmaElettronica` | string | | |
| `IDDocumentoFirmato` | GUID | | |
| `IDFirmaElettronica` | GUID | | |
| `CodiceCIG` | string | | |
| `CodiceCUP` | string | | |
| `Causale` | enum | | `"Nuovo Contratto"`, `"Modifica Contratto"`. |
| `IDContratto` | GUID | | FK verso [Contratto](#59-contratto). |
| `LegaleRappresentante` | string | | |
| `CodiceFiscaleLegaleRappresentante` | string | | |
| `NoteDelivery` | string | | |
| `DataEvasionePrevista` | date | | |

### 5.24 Opportunita

Opportunità commerciali in corso o chiuse.

**Endpoint:** `/CleverCRM/Opportunita`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | | FK verso [Cliente](#45-cliente). |
| `IDUtenteStaff` | GUID | | FK verso [Utente](#420-utente). |
| `IDUtenteUltimaModifica` | GUID | | FK verso [Utente](#420-utente). |
| `StatoAvanzamento` | enum | | `"Primo Contatto"`, `"Inviata Proposta"`, `"Contrattazione"`, `"Proposta Chiusa Positivamente"`, `"Proposta Chiusa Negativamente"`. |
| `Oggetto` | string | sì | |
| `DataApertura` | datetime | sì | |
| `DataChiusura` | datetime | | |
| `RiferimentoOpportunita` | string | sì | Codice univoco dell'opportunità (es. `COM-003284`). |
| `DataUltimaModifica` | datetime | sì | |
| `Note` | string | | |
| `Esito` | string | | |
| `StatoLavorazione` | enum | | `"Aperto"`, `"In Lavorazione"`, `"Chiuso"`, `"Sospeso"`. |

### 5.25 Ordine

Ordini di vendita generati da un'offerta.

**Endpoint:** `/CleverCRM/Ordine`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDCliente` | GUID | sì | FK verso [Cliente](#45-cliente). |
| `IDOfferta` | GUID | sì | FK verso [Offerta](#523-offerta). |
| `IDContratto` | GUID | | FK verso [Contratto](#59-contratto). |
| `Evaso` | boolean | | |
| `DataCreazione` | datetime | sì | |
| `Descrizione` | string | sì | |
| `CodiceCIG` | string | | |
| `CodiceCUP` | string | | |
| `Riferimento` | integer | | |
| `DataEvasionePrevista` | date | | |
| `DataEvasione` | date | | |

### 5.26 PianoDiFatturazione

Piano di fatturazione di una riga di dettaglio contratto.

**Endpoint:** `/CleverCRM/PianoDiFatturazione`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDDettaglioContratto` | GUID | | FK verso [DettagliContratti](#510-dettaglicontratti). |
| `DataFatturazione` | date | sì | |
| `Quantita` | number | | |
| `ImportoUnitario` | number | sì | |
| `Fatturato` | boolean | | |
| `IDFattura` | GUID | | FK verso [Fatture](#514-fatture). |
| `ScontoUnitario` | number | | |

### 5.27 PrefissoTelefonico

Prefissi telefonici associati a un gruppo.

**Endpoint:** `/CleverCRM/PrefissoTelefonico`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Prefisso` | string | sì | |
| `Descrizione` | string | sì | |
| `Regione` | string | | |
| `IDGruppoPrefissi` | GUID | | FK verso [GruppoPrefissi](#515-gruppoprefissi). |

### 5.28 SchedaTecnicaArticolo

Schede tecniche associate a un articolo.

**Endpoint:** `/CleverCRM/SchedaTecnicaArticolo`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDArticolo` | GUID | sì | FK verso [Articolo](#51-articolo). |
| `Tipo` | string | sì | Sempre `"Scheda Tecnica"`. |
| `ContentBytes` | string (Base64) | sì | Contenuto binario della scheda tecnica. |
| `ContentType` | string | sì | MIME type. |
| `Nome` | string | sì | |

---

## 6. Endpoint — CleverPresence

### 6.1 OrarioDiLavoro

Orari di lavoro configurabili da assegnare ai dipendenti.

**Endpoint:** `/CleverPresence/OrarioDiLavoro`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `Descrizione` | string | sì | |
| `OraInizio` | time | sì | Formato `HH:mm:ss`. |
| `OraFine` | time | sì | Formato `HH:mm:ss`. |
| `OraInizioPausa` | time | | |
| `OraFinePausa` | time | | |
| `Default` | boolean | sì | |

### 6.2 Presenza

Registrazione di una presenza/assenza giornaliera.

**Endpoint:** `/CleverPresence/Presenza`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | sì | FK verso [Utente](#420-utente). |
| `IDOrarioDiLavoro` | GUID | | FK verso [OrarioDiLavoro](#61-orariodilavoro). |
| `Tipo` | enum | | `"Presenza"`, `"Ferie"`, `"Malattia"`, `"Non Lavorativo"`. |
| `Data` | date | sì | |
| `Ore` | time | | Ore lavorate, formato `HH:mm:ss`. |
| `Trasferta` | boolean | | |
| `DescrizioneTrasferta` | string | | |
| `SmartWorking` | boolean | | |

### 6.3 RichiestaAssenza

Richieste di ferie / permesso / malattia presentate da un utente.

**Endpoint:** `/CleverPresence/RichiestaAssenza`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | sì | FK verso [Utente](#420-utente). |
| `Descrizione` | string | | |
| `DataDa` | date | sì | |
| `DataA` | date | sì | |
| `Tipo` | enum | | `"Ferie"`, `"Malattia"`, `"Ore di Permesso"`. |
| `Conferma` | enum | | `"In Sospeso"`, `"Approvato"`, `"Non Approvato"`. |
| `OreRichieste` | string | | |

### 6.4 Straordinario

Registrazione di ore di straordinario.

**Endpoint:** `/CleverPresence/Straordinario`

| Campo | Tipo | Obbl. | Note |
|---|---|---|---|
| `ID` | GUID | | |
| `IDUtente` | GUID | | FK verso [Utente](#420-utente). |
| `Descrizione` | string | | |
| `Data` | date | | |
| `Ore` | number | | |

---

## 7. Esempi end-to-end

Tutti gli esempi assumono che `https://{host}` punti all'installazione CleverSuite e che le credenziali siano valide.

### 7.1 Lettura di un cliente per ID

```bash
curl -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     https://{host}/CleverBase/Cliente/BA4A7337-6E46-474D-9DDD-44E76F5D09F5
```

Risposta `200 OK`:

```json
{
    "ID": "BA4A7337-6E46-474D-9DDD-44E76F5D09F5",
    "RagioneSociale": "Azienda",
    "Tipo": 30,
    "Citta": "Roma",
    "...": "..."
}
```

### 7.2 Ricerca clienti per ragione sociale

```bash
curl -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     "https://{host}/CleverBase/Cliente?RagioneSociale=Azienda"
```

Risposta `200 OK`:

```json
{
    "Cliente": [
        { "ID": "BA4A7337-...", "RagioneSociale": "Azienda", "...": "..." }
    ]
}
```

### 7.3 Creazione di un ticket

```bash
curl -X POST \
     -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     -H "Content-Type: application/json" \
     -d '{
           "IDReparto": "9C4EAF9A-54E7-4F73-8B8B-AEBB428E658E",
           "IDCliente": "EFFB35B4-9A1D-4560-B746-4DD9B0DDA877",
           "Oggetto": "Problema stampante",
           "DataApertura": "2026-05-07 10:00:00",
           "DataUltimaModifica": "2026-05-07 10:00:00",
           "RiferimentoTicket": "TCK-228000",
           "StatoTicket": "Aperto",
           "Priorita": "Media"
         }' \
     https://{host}/CleverBase/Ticket
```

Risposta `200 OK` con l'ID nel body in plain text, oppure `204 No Content`.

### 7.4 Aggiornamento di un'attività

```bash
curl -X PUT \
     -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     -H "Content-Type: application/json" \
     -d '{
           "ID": "7195FB7D-82FE-4FDA-819C-9A11933507E1",
           "Effettuato": -1,
           "DescrizioneBreve": "Problema risolto"
         }' \
     https://{host}/CleverBase/Attivita
```

Risposta `204 No Content`.

### 7.5 Eliminazione di un allegato

```bash
curl -X DELETE \
     -H "username: utente@dominio.it" \
     -H "password: la-mia-password" \
     https://{host}/CleverBase/Allegato/5C404CC1-FD82-4B94-B6A1-D9D593376169
```

Risposta `204 No Content`.
