# Steuer-App für Einzelunternehmer (tracker54) — was es ist und wie es funktioniert

Dokument fürs Portfolio: zuerst kurz zum Produkt, dann der gesamte Funktionsumfang als Liste,
danach ein zusammenhängender Weg durch die App mit Codeausschnitten und den Verbindungen dazwischen.

Code wird **ausschnittsweise** zitiert, nicht vollständig: Ziel ist es, die Logik zu zeigen,
nicht das Produkt vollständig offenzulegen.

---

## 1. Was es ist und wofür

Eine mobile App für russische Einzelunternehmer (IP) im vereinfachten Steuersystem (USN).

Ein Einzelunternehmer muss dort selbst die Steuer, die eigenen Pflichtbeiträge, die Lohnsteuer
für Mitarbeiter berechnen und darf keine Zahlungsfristen verpassen. Üblicherweise passiert das
in Excel mit Erinnerungen im Handy-Kalender.

Die App bündelt das an einem Ort: Buchungen → Berechnung von Steuer und Beiträgen →
Fristenkalender mit Erinnerungen. Alle Berechnungen laufen **auf dem Gerät**, die Cloud wird
nur für Login und Synchronisation zwischen Geräten gebraucht.

Technisch: **Expo 54 / React Native / TypeScript**, lokale Datenbank **SQLite**,
Cloud — **Supabase** (Auth + PostgreSQL). Ein Code für iOS, Android und Web.

Status: privates Projekt/Prototyp, kein fertiges Produkt aus einem App Store.

---

## 2. Der gesamte Funktionsumfang

Kurz, Punkt für Punkt. Details dazu in Abschnitt 3.

**Besteuerungsmodi**
- USN 6 % („Einnahmen“).
- USN 15 % („Einnahmen minus Ausgaben“, Mindeststeuer 1 %).
- Einrichtungsassistent beim ersten Start: Beitragstarif, Unfallversicherungssatz, USN-Satz je Jahr.

**Buchungen**
- Einnahmen und Ausgaben: Betrag, Datum, Beschreibung, Typ.
- Ausschluss eines Zahlungseingangs aus der USN-Bemessungsgrundlage (Darlehen, Rückzahlung,
  Kaution, Vermittlungsgeschäft, Sonstiges).
- Bei USN 15 %: Beleg-Foto + Auslesen des QR-Codes vom Kassenbon, Ausgabe zu einer Einnahme
  („Selbstkosten“), Anlagevermögen mit planmäßiger Abschreibung.
- Jahreswechsel, Buchungsliste, Bearbeiten und Löschen.

**Zusammenfassung auf dem Startbildschirm**
- Einnahmen in der USN-Bemessungsgrundlage und Eingänge außerhalb davon.
- Ausgaben, Gehaltszahlungen.
- Feste Pflichtbeiträge, zusätzlicher 1-%-Beitrag.
- USN-Steuer unter Berücksichtigung von Abzügen (bzw. Mindeststeuer 1 % bei USN 15 %).
- Summe „zu zahlen“ und Nettogewinn.

**Beiträge**
- Erfassung tatsächlicher Zahlungen: feste Beiträge, zusätzlicher 1 %, USN-Steuer, Sozialabgaben
  aus Gehältern.
- Wie viel berechnet, wie viel gezahlt, wie viel für das Jahr offen ist.
- Quartale und USN-Vorauszahlungen.

**Mitarbeiter**
- Mitarbeiterkarten, Beschäftigungszeitraum, individueller Unfallversicherungssatz.
- Gehaltszahlungen (Abrechnungszeitraum + Auszahlungsdatum).
- Berechnung der Lohnsteuer nach progressivem Tarif, der Sozialabgaben nach gewähltem Tarif,
  der Unfallversicherung.
- Markierungen „ans Budget überwiesen“ pro Monat, getrennt für Lohnsteuer (zwei Zeiträume),
  Sozialabgaben, Unfallversicherung.
- Monatsübersicht.

**Zahlungskalender**
- Fristen: USN-Vorauszahlungen, feste Beiträge, zusätzlicher 1 %, Lohnsteuer, Sozialabgaben,
  Unfallversicherung.
- Verschiebung der Frist auf den nächsten Werktag, Status „überfällig / bald fällig / bezahlt /
  verspätet bezahlt“.
- Lokale Erinnerungen: 7, 3, 1 Tag vorher und am Fälligkeitstag (nur auf dem Smartphone, nicht
  im Browser).

**Steuererklärung USN 6 %**
- Steuerpflichtigen-Profil: Steuernummer (INN), zuständiges Finanzamt, OKTMO-Code, vollständiger
  Name, Vertreterdaten.
- Export als XML im Format 5.09, Kodierung windows-1251, für das Online-Portal der russischen
  Steuerbehörde.

**Konto und Daten**
- Nutzung ohne Login: nur auf diesem Gerät.
- Login per E-Mail/Passwort, Registrierung, Passwort zurücksetzen.
- Cloud-Synchronisation: Änderungswarteschlange, Abholen von anderen Geräten.
- Datenisolierung bei Kontowechsel, weiches Löschen von Datensätzen.

**Darstellung**
- Heller/dunkler Modus, Tabs, Einstellungsbildschirm, Daten zurücksetzen.

---

## 3. Wie es funktioniert: der Weg durch die App

Die Reihenfolge folgt dem Weg der Nutzerin/des Nutzers: Buchung → Berechnung → Zahlung →
Frist → Mitarbeiter → Speicherung und Cloud.

### Schritt 1. Die Buchung — die einzige Quelle für alle Geldbeträge

Alles beginnt mit einem Buchungsdatensatz. Sein Modell (`lib/finance.ts`):

```ts
export interface IncomeTicket {
  id: string;
  date: string;
  amount: number;
  description: string;
  type?: OperationType;              // 'income' | 'expense'
  excludedFromUsn?: boolean;         // Einnahme vorhanden, aber nicht in der USN-Basis
  exclusionReason?: IncomeExclusionReason;
  receipt?: ExpenseReceipt;          // Foto/QR-Code des Belegs (USN 15 %)
  pairedExpense?: PairedExpense;     // Selbstkosten zu einer Einnahme
  capitalExpense?: CapitalExpenseMeta; // Anlagevermögen: planmäßige Abschreibung
  forceImmediateExpense?: boolean;
}
```

Erstellung — auf dem Tab „Учёт“ (Buchungen) (`app/(tabs)/index.tsx`):

```ts
const handleAddTicket = async () => {
  const amount = Math.round(Number(inputAmount.replace(',', '.')));
  if (!amount) return;
  if (!isAppDateAllowed(dateObject)) {
    Alert.alert('Дата', minAppDateAlertMessage());
    return;
  }
  const next: IncomeTicket = {
    id: ticketId,
    date: formatDateForStorage(dateObject),
    amount,
    description: inputDescription.trim() || 'Доход без описания',
    type: inputType,
    excludedFromUsn: excluded || undefined,
    exclusionReason: excluded ? inputExclusionReason : undefined,
  };
  await persistTickets([next, ...tickets]);
};
```

Zwei wichtige Entscheidungen sind hier direkt sichtbar:

- der Betrag wird auf ganze Rubel gerundet, Kopeken werden im Modell nicht gespeichert;
- das Datum ist nach unten begrenzt (`isAppDateAllowed`): Sätze und Tarife in der App gelten
  erst ab 2025.

Zentraler Punkt für das Verständnis der gesamten Logik: **eine Buchung ist keine Zahlung an
das Budget.** Sie verändert nur die steuerliche Bemessungsgrundlage. Zahlungen werden separat
erfasst (Schritt 3).

Was als steuerpflichtige Einnahme zählt, ist eine eigene Funktion — nicht „alle Datensätze
vom Typ income“:

```ts
export function isUsnTaxableIncomeTicket(ticket: IncomeTicket): boolean {
  return isIncomeTicket(ticket) && !isExcludedFromUsnIncome(ticket);
}
```

So bleibt ein Darlehen oder eine Rückzahlung eines Lieferanten in der Liste und im Umsatz
sichtbar, wird aber nicht besteuert.

### Schritt 2. Startbildschirm: woraus sich die Summen zusammensetzen

Alle Zahlen im Block „Ergebnisse für den Zeitraum“ werden bei jedem Rendern neu aus Buchungen
und Zahlungen berechnet (`app/(tabs)/index.tsx`, Memo `yearTotals`). Nichts wird als
„fertig berechnet“ gespeichert.

**Jahreseinnahmen** — Summe der Buchungen, die in die USN-Basis fallen:

```ts
export function getAnnualGrossIncome(tickets: IncomeTicket[], year: number): number {
  return tickets
    .filter((t) => {
      const d = parseTicketDate(t.date);
      return isUsnTaxableIncomeTicket(t) && getYearFromDate(d) === year;
    })
    .reduce((sum, t) => sum + t.amount, 0);
}
```

**Feste Pflichtbeiträge** — kommen nicht aus Buchungen, sondern sind eine Jahreskonstante:

```ts
export const FIXED_CONTRIBUTION_GOAL: Record<number, number> = {
  2026: 57_390,
  2025: 53_658,
};
```

**Zusätzlicher 1-%-Beitrag** — vom Betrag über dem Schwellenwert:

```ts
export const ADDITIONAL_CONTRIBUTION_THRESHOLD = 300_000;

export function getAdditionalContributionDue(grossIncome: number): number {
  if (grossIncome <= ADDITIONAL_CONTRIBUTION_THRESHOLD) return 0;
  const base = grossIncome - ADDITIONAL_CONTRIBUTION_THRESHOLD;
  return Math.max(0, Math.floor(base * 0.01));
}
```

**USN-Steuer 6 %** — berechnete Steuer minus Abzüge. Der Abzug setzt sich aus festem Beitrag,
zusätzlichem 1 % und Sozialabgaben aus Gehältern zusammen, die Obergrenze hängt davon ab, ob
Mitarbeiter vorhanden sind:

```ts
export function getUsnSixAccruedComponents(
  incomeYtd: number, year: number, hasEmployees: boolean,
  employeeInsuranceCredit: number, ratePercent = DEFAULT_USN_RATE_PERCENT,
) {
  const fixedCredit = getFixedContributionGoal(year);
  const additionalCredit = getAdditionalContributionDue(incomeYtd);
  const rawTaxSixPercent = Math.round((incomeYtd * safeRate) / 100);
  const totalCredits = fixedCredit + additionalCredit + employeeInsuranceCredit;
  const taxAfterCredits = getUsnSixAfterContributionCredits(
    rawTaxSixPercent, totalCredits, hasEmployees,
  );
  // ...
}
```

```ts
export function getUsnSixAfterContributionCredits(
  calculatedTaxSixPercent: number, contributionCredits: number, hasEmployees: boolean,
): number {
  if (calculatedTaxSixPercent <= 0) return 0;
  let payable: number;
  if (!hasEmployees) {
    payable = Math.max(0, calculatedTaxSixPercent - contributionCredits);
  } else {
    const maxCredit = calculatedTaxSixPercent * 0.5;      // Obergrenze 50 %
    payable = calculatedTaxSixPercent - Math.min(Math.max(0, contributionCredits), maxCredit);
  }
  return payable < 1 ? 0 : payable;
}
```

Hier zeigt sich die Verbindung zum Mitarbeiter-Modul: `employeeInsuranceCredit` kommt von
außen und mindert die Steuer — aber nur für Monate, die als bezahlt markiert wurden (Schritt 5).

**Summe „zu zahlen“** — Belastungen minus das, was bereits auf dem Tab „Взносы“ (Beiträge)
erfasst wurde:

```ts
const periodAccrualsTotal = dashboardFixedContribution + additionalOnePercent + usnTaxAmount;
const taxCascadeValueOwing = Math.max(0, periodAccrualsTotal - contributionsPaid);
```

Wobei `contributionsPaid` die Summe von drei Zahlungsarten für das Jahr ist:

```ts
export function getYearContributionsPaidTotal(contributions: ContributionPayment[], year: number) {
  return (
    getFixedContributionsPaidInYear(contributions, year) +
    getAdditionalContributionsPaidInYear(contributions, year) +
    getUsnContributionsPaidInYear(contributions, year)
  );
}
```

Daraus ergibt sich eine Regel, die das Verhalten der App für Einsteiger erklärt: Wird auf dem
Tab „Взносы“ nichts erfasst, entspricht „zu zahlen“ sämtlichen Steuern und Beiträgen des
Jahres — auch wenn das Geld tatsächlich schon geflossen ist.

**USN 15 %** läuft auf demselben Bildschirm über einen anderen Zweig: erst die anerkannten
Ausgaben, dann die Bemessungsgrundlage, dann der Vergleich mit der Mindeststeuer:

```ts
export function getUsn15TaxComponents(
  incomeYtd: number, expenseYtd: number, ratePercent: number, applyMinimumTax: boolean,
) {
  const taxBase = Math.max(0, incomeYtd - expenseYtd);
  const calculatedTax = taxBase > 0 ? Math.round((taxBase * safeRate) / 100) : 0;
  const minimumTax = applyMinimumTax
    ? Math.round((incomeYtd * USN15_MINIMUM_TAX_RATE_PERCENT) / 100)   // 1 % der Einnahmen
    : 0;
  const taxDue = applyMinimumTax ? Math.max(calculatedTax, minimumTax) : calculatedTax;
  return { taxBase, calculatedTax, minimumTax, taxDue, /* ... */ };
}
```

„Anerkannte Ausgaben“ sind nicht die Summe aller Ausgabenbuchungen: dazu zählen Buchungen,
Gehälter, Sozialabgaben aus Gehältern, und feste Beiträge sowie der zusätzliche 1 % des
Einzelunternehmers werden seit 2025 nach Zeitpunkt der Entstehung berücksichtigt
(`lib/usn15-tax.ts`). Die monatliche Aufschlüsselung zeigt ein eigener Tab „Списания“ (Zuordnung).

### Schritt 3. Beiträge — die tatsächliche Zahlung

Der Tab „Взносы“ (Beiträge) speichert nicht Belastungen, sondern Geld, das tatsächlich
geflossen ist. Der Zahlungstyp bestimmt, welche Belastung er ausgleicht:

```ts
export type ContributionKind = 'fixed' | 'additional' | 'usn' | 'payroll_insurance';

export interface ContributionPayment {
  id: string;
  date: string;
  amount: number;
  note?: string;
  kind?: ContributionKind;
}
```

Das Anlegen einer Zahlung funktioniert genauso einfach wie eine Buchung
(`app/(tabs)/contributions.tsx`):

```ts
const next: ContributionPayment = {
  id: `${Date.now()}`,
  date: formatDateForStorage(formDate),
  amount,
  note: noteInput.trim() || undefined,
  kind: formKind,
};
await persist([next, ...contributions]);
```

Die Zuordnung „für welches Jahr“ erfolgt über das Zahlungsdatum, nicht über den Zeitraum,
für den gezahlt wird:

```ts
export function getFixedContributionsPaidInYear(contributions: ContributionPayment[], year: number) {
  return contributions
    .filter((c) => getYearFromDate(parseTicketDate(c.date)) === year)
    .filter((c) => (c.kind ?? 'fixed') === 'fixed')
    .reduce((sum, c) => sum + c.amount, 0);
}
```

Verbindung zu Schritt 2: genau diese Summen werden in der Zeile „zu zahlen“ auf dem
Startbildschirm abgezogen. Eine Zahlung erzeugt keine neue Entität — sie tilgt eine bereits
berechnete Belastung.

### Schritt 4. Kalender — dieselben Summen, aber mit Fristen

Der Kalender berechnet nichts neu. Er übernimmt die Restbeträge aus den Schritten 2–3 und
ordnet sie Terminen zu (`lib/payment-calendar.ts`):

```ts
const fixedGoal = getFixedContributionGoal(taxYear);
const fixedPaid = getFixedContributionsPaidInYear(contributions, taxYear);
const fixedRemaining = Math.max(0, fixedGoal - fixedPaid);
```

Fristen — eigene Funktionen, mit Verschiebung auf den nächsten Werktag:

```ts
export function getFixedContributionPaymentDeadline(year: number): Date {
  return shiftToNextBusinessDay(new Date(year, 11, 31));       // 31. Dezember
}

export function getAdditionalContributionPaymentDeadline(year: number): Date {
  return shiftToNextBusinessDay(new Date(year + 1, 6, 1));      // 1. Juli des Folgejahres
}
```

Der Status einer Karte wird aus dem heutigen Datum und dem Zahlungsstatus berechnet:

```ts
export function getPaymentCalendarUrgency(deadline, isPaid, now = new Date(), paidDate) {
  if (isPaid) return isPaidAfterDeadline(paidDate, deadline) ? 'paid_late' : 'paid';
  const today = startOfDay(now).getTime();
  const due = startOfDay(deadline).getTime();
  if (due < today) return 'overdue';
  return due <= today + 7 * 24 * 60 * 60 * 1000 ? 'soon' : 'upcoming';
}
```

Auf denselben Ereignissen bauen die lokalen Push-Benachrichtigungen auf
(`lib/payment-reminders.ts`): 7, 3, 1 Tag vorher und mehrfach am Fälligkeitstag. Im Web sind
Benachrichtigungen deaktiviert.

### Schritt 5. Mitarbeiter — ein eigener Zweig, der in die Steuer zurückwirkt

Gehälter werden als Datensätze mit Abrechnungszeitraum und Auszahlungsdatum gespeichert:

```ts
export interface EmployeeSalaryRecord {
  id: string;
  employeeId: string;
  periodStart: string;
  periodEnd: string;
  amount: number;        // vor Lohnsteuer
  paymentDate: string;
}
```

Die Lohnsteuer wird nach einem progressiven Tarif kumulativ ab Jahresbeginn berechnet:

```ts
export const NDFL_PROGRESSIVE_BRACKETS = [
  { upTo: 2_400_000, rate: 0.13 },
  { upTo: 5_000_000, rate: 0.15 },
  { upTo: 20_000_000, rate: 0.18 },
  { upTo: 50_000_000, rate: 0.2 },
  { upTo: Number.POSITIVE_INFINITY, rate: 0.22 },
];
```

Für den Kalendermonat wird eine Schätzung „ans Budget“ zusammengestellt:

```ts
export function getEmployeeMonthBudgetEstimate(records, year, monthIndex0, tariffId, employees, defaultInjuryRatePercent) {
  const gross = getSalaryPayoutStatsInCalendarMonth(records, year, monthIndex0).totalAmount;
  const ndfl = getTotalNdflInCalendarMonth(records, year, monthIndex0);
  const insurance = getTotalEmployerInsuranceForInsuranceCard(records, year, monthIndex0, tariffId);
  const injury = getTotalInjuryForInsuranceCard(records, year, monthIndex0, employees, defaultInjuryRatePercent);
  return { gross, ndfl, insurance, injury, toBudget: ndfl + insurance + injury };
}
```

Die Sozialabgaben werden nach dem gewählten Jahrestarif berechnet: allgemein (30 % bis zur
Beitragsbemessungsgrenze, 15,1 % darüber), KMU-Tarif, KMU im verarbeitenden Gewerbe, IT,
Mikroelektronik — jeweils mit eigenen Sätzen.

Die Zahlung wird hier nicht als Betrag, sondern als monatliche Markierung erfasst:

```ts
export interface EmployeeMonthlyBudgetPayment {
  year: number;
  monthIndex0: number;
  paidNdflFirstPeriod?: boolean;   // Auszahlungen vom 1. bis 22.
  paidNdflSecondPeriod?: boolean;  // Auszahlungen ab dem 23.
  paidInsurance: boolean;
  paidInjury?: boolean;
}
```

Und hier die Verbindung zurück zu Schritt 2 — in den USN-Abzug fließen nur markierte Monate:

```ts
export function getEmployeeInsuranceCreditedThroughQuarter(records, payments, year, quarter, tariffId) {
  const lastMonthIndex0 = quarter * 3 - 1;
  let sum = 0;
  for (let m = 0; m <= lastMonthIndex0; m++) {
    const flags = getEmployeeMonthlyBudgetFlags(payments, year, m);
    if (!flags.paidInsurance) continue;                 // nicht markiert — mindert die Steuer nicht
    sum += getEmployeeMonthBudgetEstimate(records, year, m, tariffId).insurance;
  }
  return Math.round(sum);
}
```

Das heißt: die Markierung im Mitarbeiter-Modul verändert direkt die Steuersumme auf dem
Startbildschirm und erzeugt zugleich die Lohnsteuer-/Beitragstermine im Kalender.

### Schritt 6. Speicherung: SQLite und die Sendewarteschlange

Das lokale Schema (`lib/db/schema.ts`) enthält neben den fachlichen Feldern drei technische:

```sql
CREATE TABLE IF NOT EXISTS income_tickets (
  id TEXT PRIMARY KEY,
  date TEXT NOT NULL,
  amount INTEGER NOT NULL,
  -- ...
  is_dirty INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  deleted_at TEXT
);
```

- `is_dirty` — „lokal geändert, noch nicht in der Cloud angekommen“;
- `updated_at` — Zeitpunkt der letzten Änderung, danach werden Konflikte aufgelöst;
- `deleted_at` — weiches Löschen.

Jeder Schreibvorgang markiert den Datensatz als „dirty“ und stößt eine Synchronisation an
(`lib/repository/sqlite-repository.ts`):

```ts
function afterWrite(): void {
  requestPaymentRemindersReschedule();
  requestSyncIfAuthed();
}
```

Löschen entfernt die Zeile nicht, sondern setzt eine Markierung:

```ts
async function softDeleteMissingIds(table: string, keepIds: Set<string>): Promise<void> {
  const rows = await db.getAllAsync<{ id: string }>(`SELECT id FROM ${table} WHERE deleted_at IS NULL`);
  for (const row of rows) {
    if (!keepIds.has(row.id)) {
      await db.runAsync(
        `UPDATE ${table} SET deleted_at = ?, is_dirty = 1, updated_at = ? WHERE id = ?`,
        [now, now, row.id],
      );
    }
  }
}
```

Das ist nötig, damit das Löschen auch an die Cloud übermittelt werden kann. Sonst würde der
Datensatz beim nächsten Laden von einem anderen Gerät wieder auftauchen.

### Schritt 7. Cloud: eine Warteschlange, kein Warten auf Antwort

Der Bildschirm wartet nie auf den Server. Zuerst wird in SQLite geschrieben, danach folgt ein
verzögertes Senden (`lib/sync/request-sync.ts`):

```ts
const DEBOUNCE_MS = 500;

function scheduleSync(userId: string, mode: SyncMode): void {
  if (debounceTimer) clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    void runSync(id, syncMode).then((result) => {
      if (result.skipped) scheduleSync(id, syncMode);   // Sync war belegt — erneut versuchen
    });
  }, DEBOUNCE_MS);
}

/** Nach einer lokalen Änderung nur push (ohne pull, um die UI nicht zurückzusetzen). */
export function requestSync(userId: string): void {
  scheduleSync(userId, 'push');
}
```

Gesendet wird nur, was als „dirty“ markiert ist, und die Markierung wird erst nach Erfolg
zurückgesetzt (`lib/sync/push-dirty.ts`):

```ts
const dirtyTickets = await db.getAllAsync<Record<string, unknown>>(
  'SELECT * FROM income_tickets WHERE is_dirty = 1',
);
// ... Upsert in Paketen von 100 Zeilen ...
async function clearDirty(table: string, where: string, params) {
  await getDb().runAsync(`UPDATE ${table} SET is_dirty = 0 WHERE ${where}`, params);
}
```

Die Gegenrichtung ist ein Pull über den Cursor `updated_at`, mit Schutz für lokal noch nicht
gesendete Änderungen (`lib/sync/apply-sqlite.ts`):

```ts
export function shouldApplyRemoteRow(local: LocalMeta | null, remoteAt: string): boolean {
  if (!local) return true;
  if (local.is_dirty === 1) return false;             // eigene Änderung noch offen — nicht überschreiben
  return remoteIsNewer(remoteAt, local.updated_at);   // sonst gewinnt die neuere Version
}
```

Ein Löschvorgang kommt als normale Zeile mit `deleted_at` an:

```ts
if (deletedAt) {
  await getDb().runAsync(
    `UPDATE income_tickets SET deleted_at = ?, is_dirty = 0, updated_at = ? WHERE id = ?`,
    [deletedAt, remoteAt, id],
  );
  return true;
}
```

Der vollständige Zyklus (push + pull, Erstmigration beim ersten Login) ist in
`lib/sync/run-sync.ts` gebündelt, parallele Aufrufe werden über einen Mutex abgefangen.

### Schritt 8. Login: wozu die Cloud überhaupt gebraucht wird

Ohne konfiguriertes Supabase läuft die App rein lokal — der Login-Bildschirm wird gar nicht
angezeigt (`components/AuthGate.tsx`):

```tsx
if (!authEnabled) {
  return <>{children}</>;
}
if (!initialized) {
  return <ActivityIndicator />;      // Sitzung wird wiederhergestellt
}
if (!session) {
  return <AuthSignInScreen />;
}
return <>{children}</>;
```

Nach dem Login startet die Synchronisation, zusätzlich wird auf die Rückkehr der App in den
Vordergrund reagiert (`components/SyncBootstrap.tsx`):

```ts
const syncTask = (async () => {
  const wiped = await ensureFinanceDataForUser(userId);   // Kontowechsel — lokale Daten leeren
  if (wiped) notifySyncDataChanged();
  await flushSyncNow(userId);                             // vollständiger Zyklus push + pull
})();

const onAppState = (state: AppStateStatus) => {
  if (state === 'active') requestPullSync(userId);        // App wieder aktiv — abholen
};
```

In der Cloud enthält jede Tabelle eine `user_id`, der Zugriff wird über Row-Level-Security-
Policies eingeschränkt. Die Policies werden per Schleife über die Tabellenliste erzeugt, damit
die Regel überall identisch ist (`supabase/schema.sql`):

```sql
alter table public.income_tickets enable row level security;
-- ...
foreach tbl in array array['income_tickets', 'contribution_payments', 'employees', /* ... */]
loop
  execute format(
    'create policy %I_select on public.%I for select using (auth.uid() = user_id)',
    tbl || '_own', tbl
  );
  -- analog für insert (with check) und update
end loop;
```

Der Client-Key (`anon`) gewährt nur Zugriff auf Zeilen der eigenen Nutzerin/des eigenen
Nutzers: er selbst öffnet nichts — entscheidend ist `auth.uid()` in der Policy.

---

## 4. Zusammenhang im Überblick

```
      Tab „Учёт“ (Buchungen)          Tab „Взносы“ (Beiträge)
                                       (tatsächliche Zahlungen)
             │                                │
             ▼                                ▼
   USN-Basis, Ausgaben  ───►  Berechnungen in lib/  ◄───  im Jahr gezahlt
                                    │
                 ┌──────────────────┼───────────────────┐
                 ▼                  ▼                   ▼
          Startbildschirm       Kalender            Steuererklärung USN 6 %
          (Steuer, zu zahlen)   (Fristen, Push)      (XML fürs Finanzamt)
                 ▲
                 │  Sozialabgaben zählen nur bei entsprechender Markierung
          Tab „Сотрудники“ (Mitarbeiter: Gehalt, Lohnsteuer, Beiträge, Unfallvers.)

  Jeder Schreibvorgang:  UI → Repository → SQLite (is_dirty = 1)
                                       │
                                       ▼  nach 500 ms, falls eingeloggt
                                   Supabase (push), Pull über updated_at
```

Regeln, die fast das gesamte Verhalten der App erklären:

1. Eine Buchung verändert die **Bemessungsgrundlage**, eine Zahlung tilgt eine **Belastung**,
   der Kalender zeigt eine **Frist**.
2. Nichts wird fertig berechnet gespeichert: Summen werden bei jedem Rendern aus Buchungen und
   Zahlungen neu ermittelt.
3. Die lokale Datenbank ist die Wahrheitsquelle auf dem Gerät, die Cloud ist eine Kopie für den
   Login auf einem anderen Gerät.
4. Löschen bedeutet `deleted_at`, nicht das Verschwinden der Zeile.

---

## 5. Grenzen und offene Punkte

Ehrlich zum aktuellen Stand des Projekts:

- lokale Daten sind nicht verschlüsselt, es gibt keine PIN/Biometrie;
- beim ersten Login können lokale Daten die Cloud-Daten überschreiben (kein Auswahldialog);
- eine Abo-Tabelle (`subscriptions`) existiert im Schema, aber es gibt noch keine
  serverseitige Rechteprüfung;
- einige Module sind groß und sollten aufgeteilt werden;
- automatisierte Tests decken die Berechnungen ab (Skripte `scripts/verify-*.mjs`), UI und
  Synchronisation dagegen nicht;
- die Web-Version ist eingeschränkt: Belege, Kamera und Benachrichtigungen sind nur auf dem
  Smartphone verfügbar.

Das ist ein Prototyp, der bis zur Nutzbarkeit für den realen Anwendungsfall eines einzelnen
Einzelunternehmers entwickelt wurde — kein fertiges Boxprodukt.
