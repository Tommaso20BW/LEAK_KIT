# 🚨 LeakKit JR

Monitor Telegram per nuovi asset dello store Juventus e nuove pubblicazioni su Footy Headlines.

Il bot esegue tre controlli indipendenti:

1. cifre dei font di personalizzazione delle maglie 2026/27;
2. immagini fronte/retro dei prodotti Juventus 2026/27;
3. articoli Juventus nuovi o realmente aggiornati su Footy Headlines.

## Cosa controlla

### Font maglie

Per `HOME-26-27`, `AWAY-26-27` e `THIRD-26-27`, `check.py` prova le cifre da 0 a 9 sull’URL di personalizzazione dello store.

Una risposta viene accettata soltanto se:

- ha stato HTTP 200;
- dichiara un contenuto immagine non SVG;
- contiene più di 500 byte.

Quando trova un set, il bot invia un avviso e ogni cifra disponibile come foto. Poi crea un flag `.found-font-<KIT>` per non ripetere la notifica.

### Prodotti

Il bot controlla i codici `01`–`09` definiti in `PRODUCTS`: replica, authentic, maniche lunghe e portiere. Per ogni codice verifica il fronte e il retro (`_d`) in formato WebP.

Alla prima disponibilità invia le immagini trovate e crea `.found-product-<CODICE>`. Ogni prodotto resta indipendente dagli altri.

### Notizie Footy Headlines

Il bot legge la pagina Juventus di Footy Headlines e apre ogni articolo candidato per estrarre i metadati `NewsArticle`:

- titolo;
- descrizione;
- data di pubblicazione e modifica;
- fingerprint SHA-256 della versione.

In questo modo può distinguere una nuova notizia da un aggiornamento reale. Tiene sotto controllo anche gli URL già salvati, rileva vecchi URL ripubblicati e limita le novità normali a una finestra di **2 giorni**.

Lo stato è salvato in `.seen_news.json` (massimo 300 articoli). Alla prima inizializzazione gli articoli storici vengono registrati senza invii in massa.

## Stato persistente

I file seguenti impediscono notifiche duplicate:

```text
.found-font-*
.found-product-*
.seen_news.json
```

Il workflow li committa e li pubblica dopo ogni esecuzione. Per riarmare un singolo controllo di font o prodotto, elimina soltanto il relativo flag. Non cancellare `.seen_news.json` senza considerare che il bot dovrà ricostruire la baseline degli articoli.

## GitHub Actions

Il workflow [`.github/workflows/check.yml`](.github/workflows/check.yml):

- è avviabile manualmente con `workflow_dispatch`;
- usa Python 3.12;
- installa `requests` e `beautifulsoup4` direttamente nel job;
- esegue `python check.py`;
- committa i file di stato quando cambiano.

Nel repository non è presente un trigger `schedule` o `repository_dispatch`. Per controlli periodici occorre aggiungere uno schedule oppure chiamare via API il `workflow_dispatch` di `check.yml`.

## Configurazione

Configura in **Settings → Secrets and variables → Actions**:

| Secret | Obbligatorio | Uso |
|---|---:|---|
| `TELEGRAM_BOT_TOKEN` | sì | Token del bot Telegram. |
| `TELEGRAM_CHAT_ID` | sì | Chat o canale di destinazione. |

Il workflow espone anche `GIST_TOKEN`, ma `check.py` non lo legge: non è necessario al funzionamento attuale.

Il job richiede `contents: write` per aggiornare i file di stato con `GITHUB_TOKEN`.

## Avvio

### Da GitHub

Apri **Actions → Controlla leak Juventus → Run workflow**.

### In locale

```bash
python -m pip install requests beautifulsoup4
python check.py
```

Prima dell’avvio imposta `TELEGRAM_BOT_TOKEN` e `TELEGRAM_CHAT_ID`. Non è presente un `requirements.txt`.

## Personalizzazione

In `check.py` puoi aggiornare:

- `FONT_KITS` per le stagioni o i kit futuri;
- `PRODUCTS` per i codici prodotto;
- `PRODUCT_LETTER` quando cambia la lettera usata negli URL dello store;
- `NEWS_MAX_AGE_DAYS` e `NEWS_MAX_SEEN` per la finestra e la dimensione dello stato notizie.

## Limiti noti

- Gli URL dello store e il markup di Footy Headlines non sono API pubbliche stabili e possono cambiare.
- Un asset viene considerato disponibile sulla base di tipo e dimensione del file, non tramite riconoscimento visivo.
- Gli errori di rete sui singoli asset vengono registrati e il controllo prosegue; un errore non gestito a livello generale fa fallire il job.

---

Progetto amatoriale, non affiliato con Juventus FC, Telegram o Footy Headlines.
