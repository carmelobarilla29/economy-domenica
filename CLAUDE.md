# Domenica Economy

App personale di finanze di Domenica. Un solo file: `index.html`, servito da
GitHub Pages su https://carmelobarilla29.github.io/economy-domenica/ e installato sulla
home dell'iPhone come web app.

**Prima di modificare qualsiasi cosa, leggi tutto questo file.** Molte scelte
qui dentro sembrano semplificazioni o sviste: non lo sono, sono decisioni prese
insieme a Domenica dopo averle discusse. Non "migliorarle" senza chiederglielo.

## Com'è fatta

Vanilla JS, niente build, niente dipendenze, niente framework. Tutto in
`index.html`: uno `<style>`, il markup minimo, e uno `<script>` con:

- **stato** — un oggetto `S` salvato in `localStorage` sotto `domenicaeconomy.v1`
  (nessuna migrazione da altre app)
- **ledger** — un unico registro di movimenti; i saldi sono sempre calcolati
  sommando il registro, mai memorizzati
- **viste** — funzioni `vHome`, `vMov`, `vBiz`, `vPlay`, `vDebts`, `vLoans`,
  `vStats`, `vBackup`, ognuna ritorna una stringa HTML; `render()` la inietta
- **pannelli** — le funzioni `sheet*` aprono i moduli di inserimento

Font: Onest da Google Fonts. Grafici: SVG scritto a mano, nessuna libreria.

**Design**: fondo rosa chiarissimo, testo prugna, saldo in una scheda rosa scuro e
navigazione scura in basso. I portafogli in home sono schede a due colonne.
Il logo geometrico in `icon.svg` compare nell'app e come favicon;
`apple-touch-icon.png` (180 × 180, opaco) serve per l'icona sulla home iPhone.
Sono immagini statiche, non dipendenze: l'app resta tutta in `index.html`.
Se cambia il logo, aggiornare anche il PNG e la versione dei collegamenti
nel `<head>`. Non usare un SVG incorporato come icona iPhone.

## Modello dei dati

```
S = {
  ledger: [{id, w, a, l, k, ref, d, c}],   // w=portafoglio a=importo l=etichetta
                                            // k=tipo ref=collegamento d=data c=categoria
  orders: [{id, name, date, wallet, total, ship2, ship2paid, ship2date,
            wallet2, share, arrived, arrivedDate, items:[{id,name,qty}]}],
  sales:  [{id, oid, iid, qty, price, channel, wallet, date}],
  play:   [{id, kind, amount, wallet, date, note}],
  debts:  [{id, name, total, left}],
  loans:  [{id, kind, person, total, left, wallet, date, note}],
  promos: [],   // sezione tolta, i dati vecchi restano qui dentro
  bk, pin, v
}
```

**Conti e gruppi.** I **gruppi** sono fissi nel codice (`GRUPPI`): `conto`,
`contanti`, `vinted`, `risparmi`. Sono le quattro tessere che si vedono in home.
I **conti** stanno in `S.conti` (`{id, nome, gruppo}`) e li gestisce Domenica da
Impostazioni: può avere più banche dentro `conto`, e in home ne vede la somma.
Il valore speciale `nessuno` significa "questo movimento è successo ma non tocca
i miei soldi" (il gioco fatto prima dell'app, gli ordini già pagati altrove) e
non è un conto: non deve mai finire in `S.conti`.

Su ogni riga del registro `w` è l'**id di un conto**, non del gruppo. Gli id di
partenza sono `contanti`, `conto`, `vinted`, `risparmi`, gli stessi di sempre:
per questo i dati vecchi continuano a funzionare senza toccare niente, e
rinominare "Conto" in "Revolut" lascia tutta la storia attaccata dov'era.

Una tessera in home apre il dettaglio dei conti (`sheetGruppo`) se il gruppo ne
ha più di uno, altrimenti porta dritta alla correzione saldo. `saldoGruppo(g)`
somma i conti del gruppo; `liquid()` è contanti + conto.

**`balanceAt` non elenca più i portafogli a mano**: conta tutto tranne
`nessuno`. Era il punto che ci si dimenticava ogni volta che si aggiungeva un
portafoglio, e faceva ignorare quei soldi ai grafici mentre i totali del mese
li contavano.

`normalize()` fa da rete: scarta i conti senza id, i doppioni (altrimenti i
saldi si conterebbero due volte) e l'id riservato `nessuno`, rimette il gruppo
a `conto` se è inventato, e soprattutto **ricrea qualsiasi conto citato da un
movimento ma sparito dall'elenco**, così quei soldi non spariscono dai totali.

Un conto si può togliere solo se non è usato da niente — e "usato" vuol dire
anche ordini, vendite, gioco e prestiti, non solo righe del registro
(`usiDelConto`). Il numero mostrato nell'elenco è invece `movimentiDi`, cioè i
movimenti veri, perché quello è il nome che legge lui.

**`ref`** collega una riga del registro alla sua origine: `order:<id>`,
`order2:<id>`, `sale:<id>`, `play:<id>`, `giro:<uid>`, `rata:<debtId>:<importo>`,
`loan:<id>`, `rid:<loanId>:<importo>`.
Le righe con `ref` non si modificano dai Movimenti: si cambia l'oggetto
d'origine e le righe vengono riscritte. `unpost(ref)` cancella tutte le righe
di un ref.

## Le decisioni, e perché

**Il numero grande in home sono solo contanti + conto.** Non i risparmi, non il
magazzino, non il saldo Vinted, non i soldi che deve ancora riavere. È "quanto
posso spendere adesso". Sotto compare una riga con risparmi, saldo Vinted,
soldi da riavere e debiti, e il totale complessivo, che invece li conta tutti.

**Il saldo Vinted è un portafoglio come gli altri** (`daTrasferire()`). Quando
vende, i soldi entrano lì davvero: sono suoi e contano come entrata del mese,
ma per spenderli deve prima trasferirseli sul conto, e quel trasferimento è un
**giro** normale — quindi non risulta né entrata né spesa. Per questo sta fuori
dal numero grande: prima di quel giro, quei soldi in mano non ce li ha. La
tessera in home e la casella nei Movimenti compaiono solo se il saldo non è
zero, così chi non vende non se la ritrova fra i piedi.

**Il magazzino non è patrimonio.** I soldi di un ordine sono usciti e basta,
contano come spesa. In Business `stockValue()` è in cima e si chiama **"Ancora
da vendere"**: è quanto è costata la roba invenduta, la sua parte. Si chiamava
"Da rientrare" e lo abbiamo cambiato perché sembrava un debito — Domenica aveva
già coperto le spese di un ordine e in cima leggeva ancora un numero da
recuperare, mentre la scheda dell'ordine diceva "tutto recuperato". La prova
che i conti tornano: soldi incassati in più rispetto a quanto speso, più il
valore del magazzino, fa esattamente il profitto mostrato.

**Le vendite si vedono tutte**, non solo le ultime, con il conteggio
nell'intestazione. Toccando un articolo del magazzino, o un articolo dentro la
scheda di un ordine, si apre `sheetVendite()`: ogni singola vendita di quel
prodotto con data, canale, prezzo e guadagno, più il riepilogo in cima.
Dal magazzino le vendite sono raggruppate per **nome**, quindi uniscono ordini
diversi; dalla scheda dell'ordine sono del singolo articolo. Il pulsante per
vendere compare solo quando l'articolo è uno solo e ha ancora pezzi.

**I giri tra portafogli non sono né entrate né spese.** `monthInOut` e
`catBreakdown` li escludono. `balanceAt` somma tutti e tre i portafogli proprio
perché uno spostamento interno non deve risultare una perdita — è stato un bug
vero e va tenuto d'occhio.

**Il gioco sta fuori dai soldi.** Ha la sua tessera in home. Un deposito fatto
adesso esce dal conto; per quello che ha giocato prima dell'app si usa il
portafoglio `nessuno`. Il numero che conta è il saldo da sempre, non quello del
mese: serve che resti sempre visibile.

**Gli ordini si registrano col totale, mai col prezzo per pezzo.** Domenica
inserisce quanto ha pagato in tutto e solo nome e quantità degli articoli. Il
costo per pezzo è il totale diviso i pezzi, uguale per tutti. È
volutamente approssimativo sul singolo modello ed esatto sul totale: chiedere i
prezzi unitari lo bloccava e non registrava più niente. **Non reintrodurre quel
campo.**

**Ordine e spedizione per l'Italia sono due pagamenti separati**, in due momenti
diversi, ognuno con il suo portafoglio e la sua data. Finché la spedizione non è
pagata l'ordine è "aperto" e il costo per pezzo è provvisorio. Poi passa
**in transito**, e solo quando Domenica tocca il pulsante diventa **arrivato**:
pagare non vuol dire che il pacco è a casa.

**Ordini in società** (`share`, frazione da 0 a 1): la roba resta tutta a lui e
la vende lui, il socio ha una quota sui guadagni. Quindi:
- si inseriscono gli importi **pieni** dell'ordine
- dal portafoglio esce solo `totale × share`
- costo, "da rientrare", magazzino e profitto mostrati sono **i suoi**, cioè
  già moltiplicati per `share`
- `saleSocio()` calcola quanto di ogni incasso spetta al socio, e Business
  mostra "Da dare ai soci". Quando lo paga, registra una spesa normale.

**Soldi da riavere** (`S.loans`, `vLoans`, `sheetLoan`, `sheetRepay`): soldi
suoi che devono tornargli. Due tipi nello stesso elenco, distinti da `kind`:

- `kind:"prestito"` — li ha dati a una persona. Escono dal portafoglio (riga
  `prestito`, ref `loan:<id>`) ma **non sono una spesa**: `monthInOut` e
  `catBreakdown` li escludono come i giri.
- `kind:"arrivo"` — soldi suoi fermi altrove (un rimborso, un conto di gioco).
  Non sono mai stati nei portafogli, quindi **non scrivono niente nel ledger**:
  esiste solo la voce in `S.loans`.

Quando rientrano, in tutti e due i casi, una riga `rimborso` con ref
`rid:<loanId>:<importo>` porta i soldi nel portafoglio scelto. **Non è
un'entrata**: quei soldi erano già suoi, contati in `loanLeft()`.

`loanLeft()` entra nel totale complessivo in home ma **non nel numero grande**:
i soldi che devono ancora arrivare non si possono spendere adesso.

Non si può segnare un rientro più grande di quello che resta, come per le rate.
Eliminando una voce spariscono anche i rientri collegati. Le voci registrate
prima che esistesse il tipo `arrivo` non hanno `kind` e valgono come prestiti.

**La sezione Promo non c'è più.** Questa versione nasce dalla versione di Carmelo, che non la mostra. `S.promos` è rimasto nei dati e nel backup, invisibile: i suoi dati
vecchi non sono stati cancellati e la sezione si potrebbe rimettere. "Promo"
resta fra le categorie delle entrate.

**Le correzioni di saldo non sono soldi che si muovono.** Spostano il saldo (e
quindi i grafici, che sono saldi), ma restano fuori da ogni numero del tipo
"questo mese hai guadagnato X": la pillolina in home, l'intestazione del mese
nei Movimenti, "Differenza del mese" e "Dove vanno i soldi". Quelli usano
`monthReal()`, cioè entrate + uscite senza giri e senza rettifiche. La riga
"Correzioni di saldo" resta visibile a parte nel riepilogo del mese, così non
si nasconde niente. Serviva: alla prima installazione uno mette tutti i suoi
soldi con "Correggi saldo" e la home gli diceva "+650 € questo mese".

**Le date vuote** finiscono in un gruppo "Senza data", non fanno crashare
`labMonth`. **`num()`** interpreta il punto come separatore delle migliaia
(`1.234` = 1234), perché l'utente scrive all'italiana.

**Niente `confirm()` del browser**: usa `ask()`, un dialogo interno, perché il
confirm nativo può essere bloccato in certi contesti. `bind()` impedisce il
doppio tocco, e `showErr()`/`fail()` sbloccano subito dopo un errore di
validazione, altrimenti il secondo tocco veniva ignorato.

## Cose da non fare

- **Non aggiungere il prezzo per pezzo negli ordini.** Ci abbiamo rinunciato
  apposta.
- **Non far entrare magazzino, gioco o soldi da riavere nel numero grande.**
  Nel totale complessivo sotto, invece, i soldi da riavere ci vanno.
- **Non trasformare i giri in entrate/uscite.**
- **Non introdurre un blocco che sovrascrive `S` all'avvio.** Ne è esistito uno
  (caricava i dati iniziali) ed è stato rimosso apposta: era una mina, bastava
  toccarne l'identificativo per cancellare tutto quello che Domenica aveva
  registrato.
- **Non aggiungere dipendenze, build step o framework.** Un file solo, si carica
  su GitHub e va.
- **Non usare emoji** nell'interfaccia: le icone sono SVG disegnate a mano.

## Prima di consegnare una modifica

I dati vivono solo in `localStorage`, su un solo telefono. Un errore qui non è
un bug estetico: sono i suoi soldi. Prima di dirgli che è fatto:

1. Aprire il file in un browser e provare davvero il flusso toccato
2. Controllare che i saldi non siano cambiati se non dovevano
3. Se cambi qualcosa nei calcoli, verificarlo con numeri veri

Nel repo non c'è una suite di test. Se ne aggiungi una, Playwright su Chromium
funziona bene: gli elementi hanno attributi `data-*` stabili (`data-go`,
`data-sheet`, `data-led`, `data-order`, `data-adj`…) pensati proprio per essere
agganciati.

## Indipendenza dalle altre app

Repository dedicata: https://github.com/carmelobarilla29/economy-domenica.
Questa app parte vuota. Non importare dati personali o backup di altre persone.
La chiave `domenicaeconomy.v1` è esclusiva e permanente: non cambiarla e non
leggere o modificare altre chiavi. Nessuna migrazione da `soldicamera.v1`.
Le app Pages condividono lo stesso dominio; le cartelle da sole non separano
localStorage. Anche ripristino e cancellazione devono scrivere solo questa chiave.
Non usare localStorage.clear(). Non modificare le repository di Carmelo,
Alessandro o Fernando quando lavori qui. Non copiare questa pagina sopra le loro.

## Pubblicazione

GitHub Pages pubblica dalla radice del branch main. Sono sufficienti un commit
e un push in questa repository. Un aggiornamento non deve azzerare i dati.
Nessuna dipendenza, build o framework. Il PNG dell'icona deve essere opaco.

## Comunicazione

Parla in italiano semplice. Spiega prima qualsiasi modifica che cambi i numeri.
