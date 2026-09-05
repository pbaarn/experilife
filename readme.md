Empirical Sandbox // Protocol Studio — AI Developer Context & Handover

TL;DR

De Empirical Sandbox Protocol Studio is een single-file webapplicatie (index.html) gebouwd met vanilla JavaScript en Tailwind CSS. Het stelt gebruikers in staat om persoonlijke levenskeuzes en gewoonten gestructureerd te benaderen als empirische, falsifieerbare hypothesen met timeboxes en vooraf vastgestelde kill criteria.

De data-laag is via een decoupled StorageRepository opgezet: momenteel draait deze op localStorage (met een robuuste fallback naar geheugenopslag), maar het interne datamodel is 1:1 voorbereid op een relationele PostgreSQL/Supabase database. Een AI-agent kan deze codebase direct uitbreiden, refactoren of aansluiten op Supabase zonder de domeinlogica te breken.

1. Filosofie en Domeinmodel

De app voorkomt post-hoc rationalisatie (het achteraf verplaatsen van de doelpalen wanneer een experiment ongemakkelijk wordt). Elk experiment doorloopt een strikte levenscyclus met flexibele controle:

[Draft] ──> [Active / Running] <────> [Paused] (Bevroren tijdslijn)
                   │
                   ├──> [Evaluating / Review] ──> [Completed]
                   │                                 ├── Adopted (Geïnstitutionaliseerd)
                   │                                 ├── Iterated (Branch naar v2)
                   │                                 └── Abandoned (Schuldvrij gestopt)
                   │
                   ├──> [Aborted] (Kill criteria getriggerd)
                   │
                   └──> [Restart] (Geheel herstarten vanaf Sessie 1)


Kernconcepten:

Tension Audit: De frictie of nieuwsgierigheid die het experiment start (bijv. middagmoeheid).

Falsifiable Hypothesis: Een toetsbare stelling: "Als ik [MVT] toepas gedurende [Dagen], dan zal [Meetbare parameter] verbeteren met [Delta]."

MVT (Minimum Viable Test): De meest laagdrempelige uitvoering die wél valide data oplevert.

Preset Kill Criteria: Niet-onderhandelbare stopregels die vóór dag één zijn vastgelegd om de sunk cost fallacy uit te schakelen.

Bayesian Synthesis: Vergelijking tussen de verwachting en de werkelijkheid (Reality Delta), resulterend in een definitief verdict (adopted, iterated of abandoned).

2. Technische Architectuur & Stack

De app draait momenteel bewust als een volledig zelfstandig single-file artifact:

HTML5 & Vanilla JS (ES6+): Geen bundler (Vite, Webpack) nodig. Direct uitvoerbaar in elke browser of iframe.

Styling: Tailwind CSS via CDN (cdn.tailwindcss.com) met op maat geconfigureerde ontwerptokens.

Typografie:

Newsreader (serif voor hypothesen en koppen)

Plus Jakarta Sans (sans-serif voor interface-elementen)

JetBrains Mono (monospace voor metadata, labels en schema's)

Design Tokens (Editorial Light Theme):

Canvas: #FBFBF9 (warm ivoor)

Surface: #FFFFFF (helder wit)

Borderline: #E7E6E1

Accent: #2563EB (gedoseerd blauw)

Tekst: #18181B (primair) / #666560 (secundair)

3. Data Schema & Relatiemodel

Het datamodel bestaat uit twee entiteiten: Protocols en Protocol Logs.

3.1 Protocols (protocols)

Veld

Type

Nullable

Beschrijving

id

string (UUID)

Nee

Unieke identifier (bv. proto-001 of Supabase UUID)

title

string

Nee

Korte, krachtige naam van het experiment

domain

string

Nee

Domein (Vocation & Career, Somatic & Sleep, etc.)

status

string

Nee

draft, active, evaluating, completed, aborted

observation

string

Nee

De waargenomen spanning of frictie

hypothesis

string

Nee

De formele, toetsbare hypothese

intervention

string

Nee

De daadwerkelijke Minimum Viable Test (MVT)

timebox_days

integer

Nee

Looptijd in dagen (bv. 7, 14, 21, 30, 60)

start_date

ISO string

Ja

Startdatum en tijdstip van actieve run

end_date

ISO string

Ja

Verwachte einddatum op basis van timebox

success_metric

string

Nee

Welk signaal bepaalt succes?

kill_criteria

string

Nee

Harde voorwaarden om direct te staken

verdict

string

Ja

adopted, iterated, abandoned

synthesis_notes

string

Ja

Notities over de reality delta en evaluatie

created_at

ISO string

Nee

Aanmaakdatum

3.2 Protocol Logs (protocol_logs)

Veld

Type

Nullable

Beschrijving

id

string (UUID)

Nee

Unieke identificatie van de dagelijkse check-in

protocol_id

string (UUID)

Nee

Foreign key refererend naar protocols.id

day_number

integer

Nee

Relatieve dag binnen het experiment (1 t/m N)

date

string (YYYY-MM-DD)

Nee

Kalenderdatum van invoer

status

string

Nee

adhered, modified, skipped

signal_rating

integer (1-5)

Nee

Subjectieve signaal-/tevredenheidsscore

notes

string

Ja

Kwalitatieve veldobservaties van de dag

created_at

ISO string

Nee

Tijdstip van aanmaken van de log

4. Storage Abstraction (StorageRepository)

Alle data-interacties verlopen via het object StorageRepository onderaan het script. Dit object functioneert als een Data Access Object (DAO):

init(): Leest localStorage uit, hydrateert cache en laadt mock seed-data bij een lege omgeving. Bevat een try/catch-blok met fallback naar in-memory objecten voor veilige executie binnen iframes.

getProtocols(): Retourneert alle protocollen gesorteerd op created_at (aflopend).

getProtocolById(id): Zoekt specifiek protocol op ID.

saveProtocol(protocol): Upsert-mechanisme (aanmaken of bijwerken).

deleteProtocol(id): Verwijdert protocol en ruimt gecascadeerde logs op.

getLogsByProtocol(protocolId): Haalt logs op gesorteerd op day_number.

saveLog(log): Voegt een dagelijkse check-in toe of bewerkt deze.

exportAllData() / importAllData(payload): Volledige JSON export en import utility.

5. Handleiding voor Supabase Migratie

Wanneer je als AI-ontwikkelaar de opdracht krijgt om over te schakelen op Supabase, volg dan dit stappenplan:

Stap 1: Voer SQL DDL uit in Supabase

-- Activeer UUID extensie indien nodig
create extension if not exists "uuid-ossp";

-- Tabel: protocols
create table protocols (
  id uuid primary key default uuid_generate_v4(),
  user_id uuid references auth.users(id) on delete cascade, -- Optioneel voor multi-user
  title text not null,
  domain text not null,
  status text not null check (status in ('draft', 'active', 'paused', 'evaluating', 'completed', 'aborted')),
  observation text not null,
  hypothesis text not null,
  intervention text not null,
  timebox_days integer not null,
  cadence_type text not null default 'daily', -- 'daily'|'weekdays'|'specific_days'|'interval'
  cadence_config jsonb default '{}'::jsonb,  -- bv. {"selected_days": [1, 3], "duration_value": 4, "duration_unit": "weeks"}
  start_date timestamptz,
  end_date timestamptz,
  paused_at timestamptz,                     -- Tijdstip van pauzering voor automatische tijdslijncompensatie
  success_metric text not null,
  kill_criteria text not null,
  verdict text check (verdict in ('adopted', 'iterated', 'abandoned', null)),
  synthesis_notes text default '',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Tabel: protocol_logs
create table protocol_logs (
  id uuid primary key default uuid_generate_v4(),
  protocol_id uuid references protocols(id) on delete cascade,
  day_number integer not null,
  date date not null,
  status text not null check (status in ('adhered', 'modified', 'skipped')),
  signal_rating integer check (signal_rating between 1 and 5),
  notes text default '',
  created_at timestamptz default now()
);

-- Indexen voor performante queries
create index idx_protocols_status on protocols(status);
create index idx_logs_protocol on protocol_logs(protocol_id);


Stap 2: Voeg de Supabase Client toe in index.html

Plaats de officiële JS-client in de <head>:

<script src="[https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2](https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2)"></script>


Stap 3: Herschrijf StorageRepository naar Async Supabase Calls

Vervang de methoden in StorageRepository door async operaties:

const supabaseUrl = '[https://jouw-project.supabase.co](https://jouw-project.supabase.co)';
const supabaseKey = 'jouw-anon-key';
const supabase = window.supabase.createClient(supabaseUrl, supabaseKey);

const StorageRepository = {
  async getProtocols() {
    const { data, error } = await supabase
      .from('protocols')
      .select('*')
      .order('created_at', { ascending: false });
    if (error) throw error;
    return data;
  },

  async getProtocolById(id) {
    const { data, error } = await supabase
      .from('protocols')
      .select('*')
      .eq('id', id)
      .single();
    if (error) throw error;
    return data;
  },

  async saveProtocol(protocol) {
    const { data, error } = await supabase
      .from('protocols')
      .upsert(protocol)
      .select()
      .single();
    if (error) throw error;
    return data;
  },

  async deleteProtocol(id) {
    const { error } = await supabase
      .from('protocols')
      .delete()
      .eq('id', id);
    if (error) throw error;
  },

  async getLogsByProtocol(protocolId) {
    const { data, error } = await supabase
      .from('protocol_logs')
      .select('*')
      .eq('protocol_id', protocolId)
      .order('day_number', { ascending: true });
    if (error) throw error;
    return data;
  },

  async saveLog(log) {
    const { data, error } = await supabase
      .from('protocol_logs')
      .upsert(log)
      .select()
      .single();
    if (error) throw error;
    return data;
  }
};


Let op: Werk de UI-functies (renderProtocolList, renderWorkspace, form submit handlers) bij zodat ze await gebruiken bij het aanroepen van deze methodes.

6. UI Componenten & DOM Mapping

Component

DOM ID

Verantwoordelijkheid

Catalog Sidebar

#sidebarContainer

Toont zoekbalk (#searchInput), statusfilters (#statusFilterBar) en de scrollbare lijst (#protocolList).

Main Stage

#detailContainer

Toont #emptyState indien niets geselecteerd is, anders #protocolWorkspace.

Status Bar

Bovenin #protocolWorkspace

Titel, hypothese, badges en statusacties (Initiate Run, Trigger Kill Criteria, Evaluate).

Spec Matrix

Bovenste grid in workspace

Spanning, MVT, en visueel gemarkeerde Kill Criteria.

Cadence Grid

#dayTrackerGrid

Dynamisch gegenereerde dag-knoppen (D1 t/m DN) met visuele status (adhered, modified, skipped) en rating indicators.

Bayesian Synthesis

Conditioneel blok in workspace

Notitieveld (#synthesisText) en verdict-knoppen ([data-verdict]).

Field Logs

#logsStream

Chronologische tijdlijn met kwalitatieve observaties.

Protocol Modal

#protocolModal

Formulier (#protocolForm) voor aanmaken en muteren van experimenten.

Backup Modal

#dataModal

Exporteren/importeren van ruwe JSON data.

7. Richtlijnen voor AI-ontwikkelaars bij Vervolgontwikkeling

Behoud het Single-File Paradigma (tenzij expliciet gevraagd):
Houd HTML, Tailwind setup en client-side JavaScript in één enkel bestand zolang er geen overstap naar een React/Vue/Svelte toolchain wordt gevraagd.

Behoud het Rustige 'Editorial Light' Designthema:
Geen donkere modus als standaard. Geen felle neon-kleuren, glassmorphism of zware schaduwen toevoegen. Hanteer de warme ivoren achtergrond (#FBFBF9), scherpe typografie en subtiele randen (#E7E6E1).

Geen alert() of confirm() in Productie:
Vervang resterende synchrone dialogen eventueel door in-app inline elementen of custom confirmation modals, vooral wanneer de app in iframe-omgevingen draait.

Validatie van Kill Criteria:
Laat de gebruiker nooit een protocol starten zonder expliciet gedefinieerde kill criteria. Dit is de methodologische ruggengraat van de app.

8. Alternatieve Invalshoeken & Architectuuroverwegingen

Als expert-ontwikkelaar is het waardevol om ook kritische kanttekeningen en alternatieve routes te overwegen:

Lokale SQLite / OPFS versus Supabase:
Als de app bedoeld is als strikt persoonlijke, privacy-first tool, is een cloud-database zoals Supabase wellicht overkill. Een alternatief is wa-sqlite of IndexedDB (via Dexie.js) gecombineerd met de Origin Private File System (OPFS). Hiermee blijft alle data 100% lokaal en offline beschikbaar, zonder authenticatiebeheer of serverkosten.

Component Framework Migratie (React / Svelte / Vue):
De huidige DOM-manipulatie gebruikt direct innerHTML en imperatieve event listeners. Zodra realtime data (Supabase Presence of Realtime Subscriptions) wordt toegevoegd, kan een reactieve runtime (zoals Svelte of React) de UI-synchronisatie en formulier-validatie aanzienlijk vereenvoudigen en schoner houden.

Kwantificatie vs. Kwaliteit:
De huidige opzet nodigt uit tot het loggen van een numerieke 'signal score' (1-5). Er moet voor gewaakt worden dat gebruikers niet vervallen in over-instrumentalizering (het reduceren van complexe menselijke ervaringen tot arbitraire getallen). Het behouden van voldoende ruimte voor vrije tekst en subjectieve reflectie in de veldnotities is essentieel.