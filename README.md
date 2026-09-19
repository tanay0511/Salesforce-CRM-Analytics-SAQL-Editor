<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Salesforce SAQL Editor & Output Simulator</title>
  
  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- FontAwesome 6 Icons CDN -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">

  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          colors: {
            crma: {
              brand: '#0176D3',
              brandDark: '#014486',
              brandHover: '#0b5cab',
              border: '#DDDBDA',
              surface: '#F8FAFC',
              bgEditor: '#0b1120',
              bgGutter: '#070b14',
              caret: '#38bdf8'
            }
          }
        }
      }
    }
  </script>

  <style>
    @import url('https://fonts.googleapis.com/css2?family=Fira+Code:wght@400;500;600&family=Inter:wght@400;500;600;700&display=swap');

    body {
      font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
      background-color: #f3f4f6;
    }

    .font-mono-code {
      font-family: 'Fira Code', Consolas, Monaco, monospace;
    }

    /* Native Clean Editor */
    #saqlEditorInput {
      font-family: 'Fira Code', monospace;
      font-size: 13.5px;
      line-height: 22px;
      tab-size: 2;
      background-color: #0b1120;
      color: #f8fafc;
      caret-color: #38bdf8;
      border: none;
      outline: none;
      resize: none;
      white-space: pre;
      overflow-wrap: normal;
      overflow-x: auto;
    }

    #saqlEditorInput::selection {
      background-color: #1e3a8a;
      color: #ffffff;
    }

    #lineNumbers {
      font-family: 'Fira Code', monospace;
      font-size: 13.5px;
      line-height: 22px;
      background-color: #070b14;
      color: #475569;
      user-select: none;
    }

    /* Custom Scrollbars */
    ::-webkit-scrollbar { width: 6px; height: 6px; }
    ::-webkit-scrollbar-track { background: transparent; }
    ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 3px; }
    .dark-scroll::-webkit-scrollbar-track { background: #0b1120; }
    .dark-scroll::-webkit-scrollbar-thumb { background: #334155; }
    .dark-scroll::-webkit-scrollbar-thumb:hover { background: #475569; }
  </style>
</head>
<body class="h-screen flex flex-col overflow-hidden text-slate-800">

  <!-- CRMA App Header -->
  <header class="bg-white border-b border-crma-border px-4 py-2.5 flex items-center justify-between shadow-xs z-30 flex-shrink-0">
    <div class="flex items-center space-x-3">
      <div class="w-8 h-8 rounded bg-crma-brand flex items-center justify-center text-white shadow-xs">
        <i class="fa-solid fa-code text-sm"></i>
      </div>
      <div>
        <div class="flex items-center space-x-2">
          <span class="font-bold text-slate-900 text-sm">SAQL Query Editor</span>
          <span class="bg-blue-50 text-crma-brand border border-blue-200 text-[11px] font-semibold px-2 py-0.5 rounded-full flex items-center gap-1">
            <i class="fa-solid fa-database text-[10px]"></i>
            <span id="activeDatasetBadge">opportunity</span>
          </span>
          <span class="text-xs text-slate-400 hidden sm:inline">| CRMA Query Mode</span>
        </div>
        <div class="text-[11px] text-slate-500 flex items-center space-x-2 mt-0.5">
          <span id="recordCountHeader">0 rows loaded</span>
          <span>•</span>
          <span id="engineLatency">Latency: 0ms</span>
        </div>
      </div>
    </div>

    <!-- Quick Templates, Dataset Explorer, Share, Run -->
    <div class="flex items-center space-x-2">
      <select id="templateSelector" class="bg-slate-50 border border-slate-300 rounded px-2.5 py-1.5 text-xs font-medium text-slate-700 focus:outline-none focus:ring-1 focus:ring-crma-brand max-w-[210px]" title="Load a sample query">
        <optgroup label="Opportunity Templates">
          <option value="pipeline_stage" selected>Pipeline by Stage</option>
          <option value="won_by_owner">Won Amount by Owner</option>
        </optgroup>
        <optgroup label="Account & User Templates">
          <option value="account_industry">Account: Revenue by Industry</option>
          <option value="user_quota">User: Quota Attainment</option>
          <option value="account_enterprise">Account: Enterprise Ratings</option>
          <option value="cogroup_join">Cogroup: Opportunity + Account</option>
        </optgroup>
        <optgroup label="Products & Cases">
          <option value="products">Products: High Revenue Units</option>
          <option value="cases">Cases: Resolution Time Matrix</option>
        </optgroup>
      </select>

      <button id="openSchemaBtn" class="bg-white border border-slate-300 hover:bg-slate-50 text-slate-700 font-medium px-2.5 py-1.5 rounded text-xs transition flex items-center gap-1.5" title="View available datasets and fields">
        <i class="fa-solid fa-database text-crma-brand text-xs"></i>
        <span class="hidden md:inline">Schema (5)</span>
      </button>

      <button id="exportCsvBtn" class="bg-white border border-slate-300 hover:bg-slate-50 text-slate-700 font-medium px-2.5 py-1.5 rounded text-xs transition flex items-center gap-1.5" title="Export current results table to CSV">
        <i class="fa-solid fa-file-csv text-emerald-600 text-xs"></i>
        <span class="hidden md:inline">Export CSV</span>
      </button>

      <button id="shareLinkBtn" class="bg-white border border-slate-300 hover:bg-slate-50 text-slate-700 font-medium px-2.5 py-1.5 rounded text-xs transition flex items-center gap-1.5" title="Copy sharable URL that contains your current query">
        <i class="fa-solid fa-share-nodes text-crma-brand text-xs"></i>
        <span class="hidden md:inline">Share Link</span>
      </button>

      <button id="runSaqlBtn" class="bg-crma-brand hover:bg-crma-brandDark text-white font-semibold px-3.5 py-1.5 rounded text-xs transition flex items-center gap-1.5 shadow-xs active:scale-95">
        <i class="fa-solid fa-play text-[10px]"></i>
        <span>Run Query</span>
        <kbd class="hidden lg:inline bg-blue-800/60 text-[10px] px-1 py-0.5 rounded font-mono ml-0.5">Ctrl+Enter</kbd>
      </button>
    </div>
  </header>

  <!-- Split Screen: Top Editor & Bottom Tabular Output -->
  <main class="flex-1 flex flex-col md:flex-row overflow-hidden">
    
    <!-- LEFT / TOP PANEL: Clean SAQL Editor -->
    <section class="w-full md:w-1/2 flex flex-col bg-[#0b1120] border-b md:border-b-0 md:border-r border-slate-800 h-1/2 md:h-full">
      <!-- Editor Actions Toolbar -->
      <div class="bg-[#070b14] border-b border-slate-800 px-3.5 py-2 flex items-center justify-between text-xs text-slate-300 flex-shrink-0">
        <div class="flex items-center space-x-2 font-mono">
          <span class="w-2 h-2 rounded-full bg-emerald-400 inline-block"></span>
          <span class="text-white font-medium text-xs">SAQL Statement</span>
        </div>
        <div class="flex items-center space-x-2">
          <button id="formatSaqlBtn" class="px-2 py-1 bg-slate-800/80 hover:bg-slate-700 text-slate-200 rounded text-xs transition flex items-center gap-1" title="Auto format query">
            <i class="fa-solid fa-wand-magic-sparkles text-indigo-400 text-xs"></i>
            <span>Format</span>
          </button>
          <button id="copySaqlBtn" class="px-2 py-1 bg-slate-800/80 hover:bg-slate-700 text-slate-200 rounded text-xs transition flex items-center gap-1" title="Copy SAQL text">
            <i class="fa-regular fa-copy text-xs"></i>
            <span>Copy</span>
          </button>
        </div>
      </div>

      <!-- Native Editor Body: Synchronized Gutter + Pure High-Performance Textarea -->
      <div class="relative flex-1 flex overflow-hidden bg-[#0b1120]">
        <div id="lineNumbers" class="w-10 bg-[#070b14] border-r border-slate-800 py-3 text-right pr-2 select-none font-mono-code flex-shrink-0 overflow-hidden">
          1
        </div>
        <textarea id="saqlEditorInput" spellcheck="false" autocomplete="off" autocorrect="off" autocapitalize="off" class="dark-scroll flex-1 h-full p-3 font-mono-code focus:outline-none" placeholder="Type your SAQL here..."></textarea>
      </div>

      <!-- Execution Status Bar -->
      <div id="saqlStatusBar" class="px-3.5 py-1.5 bg-[#070b14] border-t border-slate-800 text-xs flex items-center justify-between text-slate-400 flex-shrink-0">
        <div class="flex items-center space-x-2 truncate" id="saqlStatusText">
          <i class="fa-solid fa-circle-check text-emerald-400 text-xs"></i>
          <span class="text-slate-300">Ready. Press <kbd class="font-mono bg-slate-800 px-1 rounded text-slate-200">Ctrl+Enter</kbd> to execute.</span>
        </div>
        <span class="font-mono text-[11px] text-slate-500">CRMA Virtual Runtime</span>
      </div>
    </section>

    <!-- RIGHT / BOTTOM PANEL: Tabular Output Grid -->
    <section class="w-full md:w-1/2 flex flex-col bg-white h-1/2 md:h-full overflow-hidden">
      <!-- Output Header Bar -->
      <div class="bg-slate-50 border-b border-crma-border px-4 py-2 flex items-center justify-between text-xs flex-shrink-0">
        <div class="flex items-center space-x-2">
          <i class="fa-solid fa-table text-crma-brand"></i>
          <span class="font-bold text-slate-800">Query Output (Tabular)</span>
          <span class="text-slate-400">•</span>
          <span id="outputTableSummary" class="text-slate-500">0 rows returned</span>
        </div>
        <div class="text-[11px] text-slate-500 font-mono">
          Click any column header to sort
        </div>
      </div>

      <!-- Table Container -->
      <div class="flex-1 overflow-auto bg-slate-50/50 relative">
        <table class="w-full text-left text-xs border-collapse" id="resultsDataTable">
          <thead class="bg-slate-100/90 border-b border-slate-200 sticky top-0 z-10 text-slate-700 font-semibold" id="resultsTableHead">
            <tr>
              <th class="p-3 text-slate-400 font-normal">No query executed yet.</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-slate-100 font-mono text-[12px] bg-white" id="resultsTableBody">
            <tr>
              <td class="p-8 text-center text-slate-400 font-sans">
                Run a query to populate tabular results.
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

  </main>

  <!-- Toast Notification -->
  <div id="toastNotification" class="fixed bottom-4 right-4 bg-slate-900 text-white px-4 py-2.5 rounded-lg shadow-xl text-xs flex items-center space-x-2 transform translate-y-20 opacity-0 transition-all duration-300 z-50">
    <i id="toastIcon" class="fa-solid fa-circle-check text-emerald-400"></i>
    <span id="toastMessage">Done</span>
  </div>

  <!-- Dataset Schema Modal -->
  <div id="schemaModal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-xs z-50 flex items-center justify-center hidden">
    <div class="bg-white rounded-xl shadow-2xl w-11/12 max-w-4xl max-h-[85vh] flex flex-col border border-slate-200 overflow-hidden">
      <div class="px-6 py-3.5 border-b border-slate-200 flex items-center justify-between bg-slate-50">
        <div class="flex items-center space-x-2.5">
          <div class="p-2 bg-blue-100 rounded text-crma-brand">
            <i class="fa-solid fa-database text-sm"></i>
          </div>
          <div>
            <h3 class="text-sm font-bold text-slate-900">Dataset Schema Explorer</h3>
            <p class="text-xs text-slate-500">Available mock datasets for your SAQL queries</p>
          </div>
        </div>
        <button id="closeSchemaModal" class="text-slate-400 hover:text-slate-600">
          <i class="fa-solid fa-xmark text-base"></i>
        </button>
      </div>

      <div class="px-6 pt-2.5 flex space-x-2 border-b border-slate-200 bg-white overflow-x-auto">
        <button class="dataset-tab font-semibold text-xs pb-2 border-b-2 border-crma-brand text-crma-brand" data-ds="opportunity">Opportunity (12)</button>
        <button class="dataset-tab font-medium text-xs pb-2 text-slate-500 hover:text-slate-800" data-ds="Account">Account (12)</button>
        <button class="dataset-tab font-medium text-xs pb-2 text-slate-500 hover:text-slate-800" data-ds="User">User (8)</button>
        <button class="dataset-tab font-medium text-xs pb-2 text-slate-500 hover:text-slate-800" data-ds="products">Products (10)</button>
        <button class="dataset-tab font-medium text-xs pb-2 text-slate-500 hover:text-slate-800" data-ds="cases">Cases (10)</button>
      </div>

      <div class="p-6 overflow-y-auto space-y-4 flex-1 text-xs">
        <div>
          <h4 class="font-bold text-slate-600 uppercase text-[11px] mb-2">Fields & Types</h4>
          <div class="grid grid-cols-2 md:grid-cols-4 gap-2" id="schemaGridContent"></div>
        </div>
        <div>
          <h4 class="font-bold text-slate-600 uppercase text-[11px] mb-2">Sample Records</h4>
          <div class="border border-slate-200 rounded overflow-x-auto">
            <table class="w-full text-left text-xs" id="schemaSampleTable"></table>
          </div>
        </div>
      </div>

      <div class="px-6 py-3 bg-slate-50 border-t border-slate-200 flex justify-between items-center">
        <span class="text-xs text-slate-500">Foreign keys: <code class="text-crma-brand">AccountId</code> & <code class="text-crma-brand">OwnerId</code></span>
        <button id="insertDatasetLoadBtn" class="bg-crma-brand hover:bg-crma-brandDark text-white px-3 py-1.5 rounded text-xs font-semibold">Load Selected in Editor</button>
      </div>
    </div>
  </div>

  <script>
    const DATASETS = {
      Account: [
        { Id: "ACC-001", Name: "Acme Corporation", Industry: "Technology", Type: "Customer - Direct", AnnualRevenue: 45000000, NumberOfEmployees: 4200, BillingCountry: "United States", BillingState: "CA", Rating: "Hot", OwnerId: "USR-001" },
        { Id: "ACC-002", Name: "Global Tech Holdings", Industry: "Technology", Type: "Customer - Channel", AnnualRevenue: 85000000, NumberOfEmployees: 9800, BillingCountry: "United Kingdom", BillingState: "London", Rating: "Warm", OwnerId: "USR-002" },
        { Id: "ACC-003", Name: "Starlight Health Care", Industry: "Healthcare", Type: "Customer - Direct", AnnualRevenue: 120000000, NumberOfEmployees: 14500, BillingCountry: "United States", BillingState: "NY", Rating: "Hot", OwnerId: "USR-001" },
        { Id: "ACC-004", Name: "Zenith Retail Group", Industry: "Retail", Type: "Prospect", AnnualRevenue: 28000000, NumberOfEmployees: 3200, BillingCountry: "Singapore", BillingState: "Central", Rating: "Warm", OwnerId: "USR-003" },
        { Id: "ACC-005", Name: "Apex Financial Systems", Industry: "Finance", Type: "Customer - Direct", AnnualRevenue: 65000000, NumberOfEmployees: 2900, BillingCountry: "Germany", BillingState: "Frankfurt", Rating: "Cold", OwnerId: "USR-002" },
        { Id: "ACC-006", Name: "BlueCloud Media", Industry: "Technology", Type: "Customer - Direct", AnnualRevenue: 19000000, NumberOfEmployees: 1100, BillingCountry: "Australia", BillingState: "NSW", Rating: "Hot", OwnerId: "USR-003" },
        { Id: "ACC-007", Name: "Nexus Renewable Energy", Industry: "Energy", Type: "Prospect", AnnualRevenue: 92000000, NumberOfEmployees: 6100, BillingCountry: "United States", BillingState: "TX", Rating: "Hot", OwnerId: "USR-001" },
        { Id: "ACC-008", Name: "Omni Logistics & Cargo", Industry: "Manufacturing", Type: "Customer - Direct", AnnualRevenue: 54000000, NumberOfEmployees: 5300, BillingCountry: "Brazil", BillingState: "SP", Rating: "Hot", OwnerId: "USR-004" },
        { Id: "ACC-009", Name: "Delta Capital Partners", Industry: "Finance", Type: "Customer - Channel", AnnualRevenue: 110000000, NumberOfEmployees: 4800, BillingCountry: "United States", BillingState: "IL", Rating: "Warm", OwnerId: "USR-001" },
        { Id: "ACC-010", Name: "Veritas Medical Bio", Industry: "Healthcare", Type: "Prospect", AnnualRevenue: 34000000, NumberOfEmployees: 2100, BillingCountry: "France", BillingState: "IDF", Rating: "Cold", OwnerId: "USR-002" },
        { Id: "ACC-011", Name: "Solaris Power Grid", Industry: "Energy", Type: "Partner", AnnualRevenue: 78000000, NumberOfEmployees: 4700, BillingCountry: "Mexico", BillingState: "CDMX", Rating: "Hot", OwnerId: "USR-004" },
        { Id: "ACC-012", Name: "Nova Automation Labs", Industry: "Manufacturing", Type: "Prospect", AnnualRevenue: 41000000, NumberOfEmployees: 3100, BillingCountry: "Japan", BillingState: "Tokyo", Rating: "Warm", OwnerId: "USR-003" }
      ],
      User: [
        { Id: "USR-001", Username: "sconnor@enterprise.crm", FullName: "Sarah Connor", Role: "VP Sales", Department: "Sales", Region: "North America", Quota: 2500000, TargetAttainmentPct: 118.5, IsActive: true },
        { Id: "USR-002", Username: "jmiller@enterprise.crm", FullName: "John Miller", Role: "Sales Executive", Department: "Sales", Region: "EMEA", Quota: 1800000, TargetAttainmentPct: 82.3, IsActive: true },
        { Id: "USR-003", Username: "akhan@enterprise.crm", FullName: "Aisha Khan", Role: "Sales Executive", Department: "Sales", Region: "APAC", Quota: 1600000, TargetAttainmentPct: 104.0, IsActive: true },
        { Id: "USR-004", Username: "cgomez@enterprise.crm", FullName: "Carlos Gomez", Role: "Sales Executive", Department: "Sales", Region: "LATAM", Quota: 1400000, TargetAttainmentPct: 112.7, IsActive: true },
        { Id: "USR-005", Username: "emily.chen@enterprise.crm", FullName: "Emily Chen", Role: "Solutions Architect", Department: "Engineering", Region: "North America", Quota: 900000, TargetAttainmentPct: 96.0, IsActive: true },
        { Id: "USR-006", Username: "dkovacs@enterprise.crm", FullName: "David Kovacs", Role: "SDR", Department: "Marketing", Region: "EMEA", Quota: 500000, TargetAttainmentPct: 125.0, IsActive: true },
        { Id: "USR-007", Username: "patel.r@enterprise.crm", FullName: "Rohan Patel", Role: "Support Lead", Department: "Support", Region: "APAC", Quota: 400000, TargetAttainmentPct: 98.2, IsActive: true },
        { Id: "USR-008", Username: "laura.b@enterprise.crm", FullName: "Laura Bennett", Role: "SDR", Department: "Marketing", Region: "North America", Quota: 550000, TargetAttainmentPct: 78.0, IsActive: false }
      ],
      opportunity: [
        { Id: "0061", AccountId: "ACC-001", AccountName: "Acme Corporation", Stage: "Closed Won", Amount: 125000, CloseDate: "2026-03-15", Type: "Existing Customer", Region: "North America", OwnerId: "USR-001", Owner: "Sarah Connor", Probability: 1.0 },
        { Id: "0062", AccountId: "ACC-002", AccountName: "Global Tech", Stage: "Negotiation", Amount: 85000, CloseDate: "2026-04-10", Type: "New Business", Region: "EMEA", OwnerId: "USR-002", Owner: "John Miller", Probability: 0.8 },
        { Id: "0063", AccountId: "ACC-003", AccountName: "Starlight Health", Stage: "Closed Won", Amount: 240000, CloseDate: "2026-02-28", Type: "Existing Customer", Region: "North America", OwnerId: "USR-001", Owner: "Sarah Connor", Probability: 1.0 },
        { Id: "0064", AccountId: "ACC-004", AccountName: "Zenith Retail", Stage: "Prospecting", Amount: 32000, CloseDate: "2026-05-18", Type: "New Business", Region: "APAC", OwnerId: "USR-003", Owner: "Aisha Khan", Probability: 0.2 },
        { Id: "0065", AccountId: "ACC-005", AccountName: "Apex Financial", Stage: "Closed Lost", Amount: 95000, CloseDate: "2026-01-20", Type: "Existing Customer", Region: "EMEA", OwnerId: "USR-002", Owner: "John Miller", Probability: 0.0 },
        { Id: "0066", AccountId: "ACC-006", AccountName: "BlueCloud Media", Stage: "Closed Won", Amount: 45000, CloseDate: "2026-03-02", Type: "New Business", Region: "APAC", OwnerId: "USR-003", Owner: "Aisha Khan", Probability: 1.0 },
        { Id: "0067", AccountId: "ACC-007", AccountName: "Nexus Energy", Stage: "Proposal", Amount: 180000, CloseDate: "2026-04-22", Type: "New Business", Region: "North America", OwnerId: "USR-001", Owner: "Sarah Connor", Probability: 0.6 },
        { Id: "0068", AccountId: "ACC-008", AccountName: "Omni Logistics", Stage: "Closed Won", Amount: 62000, CloseDate: "2026-02-14", Type: "Existing Customer", Region: "LATAM", OwnerId: "USR-004", Owner: "Carlos Gomez", Probability: 1.0 },
        { Id: "0069", AccountId: "ACC-009", AccountName: "Delta Capital", Stage: "Negotiation", Amount: 110000, CloseDate: "2026-04-30", Type: "Existing Customer", Region: "North America", OwnerId: "USR-001", Owner: "Sarah Connor", Probability: 0.75 },
        { Id: "0070", AccountId: "ACC-010", AccountName: "Veritas Medical", Stage: "Closed Lost", Amount: 40000, CloseDate: "2026-02-05", Type: "New Business", Region: "EMEA", OwnerId: "USR-002", Owner: "John Miller", Probability: 0.0 },
        { Id: "0071", AccountId: "ACC-011", AccountName: "Solaris Power", Stage: "Closed Won", Amount: 195000, CloseDate: "2026-03-29", Type: "New Business", Region: "LATAM", OwnerId: "USR-004", Owner: "Carlos Gomez", Probability: 1.0 },
        { Id: "0072", AccountId: "ACC-012", AccountName: "Nova Robotics", Stage: "Proposal", Amount: 78000, CloseDate: "2026-05-01", Type: "New Business", Region: "APAC", OwnerId: "USR-003", Owner: "Aisha Khan", Probability: 0.5 }
      ],
      products: [
        { Id: "PRD-01", ProductName: "Analytics Cloud Unlimited", Category: "Software", UnitPrice: 1500, UnitsSold: 120, Revenue: 180000 },
        { Id: "PRD-02", ProductName: "Sales Cloud Enterprise", Category: "Software", UnitPrice: 2000, UnitsSold: 95, Revenue: 190000 },
        { Id: "PRD-03", ProductName: "Service Cloud Pro", Category: "Software", UnitPrice: 1200, UnitsSold: 150, Revenue: 180000 },
        { Id: "PRD-04", ProductName: "Edge Gateway Router", Category: "Hardware", UnitPrice: 850, UnitsSold: 60, Revenue: 51000 },
        { Id: "PRD-05", ProductName: "IoT Telemetry Sensor", Category: "Hardware", UnitPrice: 220, UnitsSold: 340, Revenue: 74800 },
        { Id: "PRD-06", ProductName: "Premier 24/7 Support", Category: "Services", UnitPrice: 5000, UnitsSold: 42, Revenue: 210000 },
        { Id: "PRD-07", ProductName: "Implementation Sprint", Category: "Services", UnitPrice: 8000, UnitsSold: 18, Revenue: 144000 },
        { Id: "PRD-08", ProductName: "Einstein Discovery AI", Category: "Software", UnitPrice: 900, UnitsSold: 80, Revenue: 72000 },
        { Id: "PRD-09", ProductName: "Data Cloud Storage Pack", Category: "Infrastructure", UnitPrice: 400, UnitsSold: 210, Revenue: 84000 },
        { Id: "PRD-10", ProductName: "Security Shield", Category: "Security", UnitPrice: 3100, UnitsSold: 35, Revenue: 108500 }
      ],
      cases: [
        { Id: "CAS-101", AccountId: "ACC-001", AccountName: "Acme Corporation", Priority: "High", Status: "Closed", CreatedDate: "2026-02-10", ResolutionTimeDays: 2.1, Department: "Technical Support" },
        { Id: "CAS-102", AccountId: "ACC-002", AccountName: "Global Tech", Priority: "Critical", Status: "Closed", CreatedDate: "2026-02-12", ResolutionTimeDays: 0.8, Department: "Engineering" },
        { Id: "CAS-103", AccountId: "ACC-003", AccountName: "Starlight Health", Priority: "Medium", Status: "Closed", CreatedDate: "2026-02-15", ResolutionTimeDays: 4.5, Department: "Billing" },
        { Id: "CAS-104", AccountId: "ACC-005", AccountName: "Apex Financial", Priority: "Low", Status: "Closed", CreatedDate: "2026-02-18", ResolutionTimeDays: 6.2, Department: "Technical Support" },
        { Id: "CAS-105", AccountId: "ACC-006", AccountName: "BlueCloud Media", Priority: "High", Status: "New", CreatedDate: "2026-03-01", ResolutionTimeDays: 1.2, Department: "Technical Support" },
        { Id: "CAS-106", AccountId: "ACC-008", AccountName: "Omni Logistics", Priority: "Critical", Status: "Closed", CreatedDate: "2026-03-02", ResolutionTimeDays: 0.5, Department: "Engineering" },
        { Id: "CAS-107", AccountId: "ACC-007", AccountName: "Nexus Energy", Priority: "Medium", Status: "In Progress", CreatedDate: "2026-03-05", ResolutionTimeDays: 3.1, Department: "Billing" },
        { Id: "CAS-108", AccountId: "ACC-009", AccountName: "Delta Capital", Priority: "High", Status: "Closed", CreatedDate: "2026-03-08", ResolutionTimeDays: 1.8, Department: "Technical Support" },
        { Id: "CAS-109", AccountId: "ACC-010", AccountName: "Veritas Medical", Priority: "Low", Status: "Closed", CreatedDate: "2026-03-10", ResolutionTimeDays: 5.0, Department: "Billing" },
        { Id: "CAS-110", AccountId: "ACC-011", AccountName: "Solaris Power", Priority: "Critical", Status: "In Progress", CreatedDate: "2026-03-12", ResolutionTimeDays: 1.1, Department: "Engineering" }
      ]
    };

    const SCHEMAS = {
      Account: {
        Id: "Dimension", Name: "Dimension", Industry: "Dimension",
        Type: "Dimension", AnnualRevenue: "Measure (Currency)",
        NumberOfEmployees: "Measure (Number)", BillingCountry: "Dimension",
        Rating: "Dimension", OwnerId: "Dimension"
      },
      User: {
        Id: "Dimension", Username: "Dimension", FullName: "Dimension",
        Role: "Dimension", Department: "Dimension", Region: "Dimension",
        Quota: "Measure (Currency)", TargetAttainmentPct: "Measure (Percent)",
        IsActive: "Dimension (Boolean)"
      },
      opportunity: {
        Id: "Dimension", AccountId: "Dimension", AccountName: "Dimension",
        Stage: "Dimension", Amount: "Measure (Currency)",
        CloseDate: "Date", Type: "Dimension", Region: "Dimension",
        OwnerId: "Dimension", Owner: "Dimension", Probability: "Measure (Percent)"
      },
      products: {
        Id: "Dimension", ProductName: "Dimension", Category: "Dimension",
        UnitPrice: "Measure (Currency)", UnitsSold: "Measure (Number)",
        Revenue: "Measure (Currency)"
      },
      cases: {
        Id: "Dimension", AccountName: "Dimension", Priority: "Dimension",
        Status: "Dimension", CreatedDate: "Date",
        ResolutionTimeDays: "Measure (Decimal)", Department: "Dimension"
      }
    };

    const TEMPLATES = {
      pipeline_stage: `q = load "opportunity";\nq = group q by 'Stage';\nq = foreach q generate 'Stage' as 'Stage', sum('Amount') as 'Pipeline', count() as 'Deals', avg('Amount') as 'Avg_Deal';\nq = order q by 'Pipeline' desc;`,
      won_by_owner: `q = load "opportunity";\nq = filter q by Stage == "Closed Won";\nq = group q by 'Owner';\nq = foreach q generate 'Owner' as 'Owner', sum('Amount') as 'Closed_Won_Amount', count() as 'Deals_Won';\nq = order q by 'Closed_Won_Amount' desc;`,
      account_industry: `q = load "Account";\nq = group q by ('Industry', 'Rating');\nq = foreach q generate 'Industry' as 'Industry', 'Rating' as 'Rating', sum('AnnualRevenue') as 'Total_Revenue', count() as 'Accounts', avg('NumberOfEmployees') as 'Avg_Emp';\nq = order q by 'Total_Revenue' desc;`,
      user_quota: `q = load "User";\nq = filter q by IsActive == true;\nq = group q by 'Region';\nq = foreach q generate 'Region' as 'Region', sum('Quota') as 'Region_Quota', avg('TargetAttainmentPct') as 'Avg_Attainment', count() as 'Reps';\nq = order q by 'Avg_Attainment' desc;`,
      account_enterprise: `q = load "Account";\nq = filter q by NumberOfEmployees > 3000;\nq = foreach q generate 'Name' as 'Company', 'Industry' as 'Industry', 'AnnualRevenue' as 'Revenue', 'NumberOfEmployees' as 'Employees', 'Rating' as 'Rating';\nq = order q by 'Revenue' desc;`,
      cogroup_join: `o = load "opportunity";\na = load "Account";\nq = cogroup o by 'AccountId' right, a by 'Id';\nq = foreach q generate a['Name'] as 'AccountName', a['Industry'] as 'Industry', coalesce(sum(o['Amount']), 0) as 'Won_Pipe', count(o) as 'Deal_Count';\nq = order q by 'Won_Pipe' desc;`,
      products: `q = load "products";\nq = foreach q generate 'ProductName' as 'Product', 'Category' as 'Category', 'Revenue' as 'Revenue', 'UnitsSold' as 'Units';\nq = order q by 'Revenue' desc;\nq = limit q 6;`,
      cases: `q = load "cases";\nq = group q by ('Priority', 'Department');\nq = foreach q generate 'Priority' as 'Priority', 'Department' as 'Department', avg('ResolutionTimeDays') as 'Avg_Days', count() as 'Case_Count';\nq = order q by 'Avg_Days' asc;`
    };
  </script>

  <script>
    class SAQLEngine {
      constructor(datasets) {
        this.datasets = datasets;
      }

      getDataset(name) {
        const lower = name.toLowerCase();
        for (const k of Object.keys(this.datasets)) {
          if (k.toLowerCase() === lower) return this.datasets[k];
        }
        return null;
      }

      execute(saqlCode) {
        const startTime = performance.now();
        const cleaned = saqlCode.replace(/\/\*[\s\S]*?\*\/|([^:]|^)\/\/.*$/gm, '').trim();
        if (!cleaned) throw new Error("SAQL editor is empty. Enter query statements.");

        const rawStatements = cleaned.split(';').map(s => s.trim()).filter(s => s.length > 0);
        if (rawStatements.length === 0) throw new Error("Ensure all statements end with a semicolon ';'.");

        const streams = {};
        let lastStream = "q";
        let detectedDataset = "opportunity";

        for (let i = 0; i < rawStatements.length; i++) {
          const stmt = rawStatements[i];
          const assignMatch = stmt.match(/^([a-zA-Z0-9_]+)\s*=\s*([\s\S]+)$/);
          if (!assignMatch) throw new Error(`Syntax Error near statement ${i + 1}: Missing variable assignment.`);

          const target = assignMatch[1].trim();
          const expr = assignMatch[2].trim();

          // LOAD
          const loadMatch = expr.match(/^load\s+["']([^"']+)["']$/i);
          if (loadMatch) {
            const dsName = loadMatch[1].trim();
            const data = this.getDataset(dsName);
            if (!data) throw new Error(`Dataset "${dsName}" not found. Check Dataset Schema for available datasets.`);
            streams[target] = { data: JSON.parse(JSON.stringify(data)), isGrouped: false };
            lastStream = target;
            detectedDataset = dsName;
            continue;
          }

          // COGROUP
          const cogroupMatch = expr.match(/^cogroup\s+([a-zA-Z0-9_]+)\s+by\s+['"]?([a-zA-Z0-9_]+)['"]?\s*(left|right|full|inner)?\s*,\s*([a-zA-Z0-9_]+)\s+by\s+['"]?([a-zA-Z0-9_]+)['"]?$/i);
          if (cogroupMatch) {
            const s1Name = cogroupMatch[1], s1Key = cogroupMatch[2];
            const joinType = (cogroupMatch[3] || 'inner').toLowerCase();
            const s2Name = cogroupMatch[4], s2Key = cogroupMatch[5];
            if (!streams[s1Name] || !streams[s2Name]) throw new Error(`Stream not found for cogroup join.`);

            const groups = new Map();
            streams[s1Name].data.forEach(row => {
              const k = String(row[s1Key] || '');
              if (!groups.has(k)) groups.set(k, { [s1Name]: [], [s2Name]: [] });
              groups.get(k)[s1Name].push(row);
            });
            streams[s2Name].data.forEach(row => {
              const k = String(row[s2Key] || '');
              if (!groups.has(k) && (joinType === 'right' || joinType === 'full')) {
                groups.set(k, { [s1Name]: [], [s2Name]: [] });
              }
              if (groups.has(k)) groups.get(k)[s2Name].push(row);
            });

            streams[target] = { isCogrouped: true, isGrouped: true, groups, s1Name, s2Name, data: [] };
            lastStream = target;
            continue;
          }

          // FILTER
          const filterMatch = expr.match(/^filter\s+([a-zA-Z0-9_]+)\s+by\s+([\s\S]+)$/i);
          if (filterMatch) {
            const src = filterMatch[1].trim(), cond = filterMatch[2].trim();
            if (!streams[src]) throw new Error(`Stream "${src}" not defined.`);
            streams[target] = {
              data: streams[src].data.filter(r => this.evalCondition(r, cond)),
              isGrouped: streams[src].isGrouped
            };
            lastStream = target;
            continue;
          }

          // GROUP
          const groupMatch = expr.match(/^group\s+([a-zA-Z0-9_]+)\s+by\s+([\s\S]+)$/i);
          if (groupMatch) {
            const src = groupMatch[1].trim();
            let dimsStr = groupMatch[2].trim();
            if (!streams[src]) throw new Error(`Stream "${src}" not defined.`);

            let dims = [];
            if (dimsStr.toLowerCase() === 'all') dims = ['__all__'];
            else {
              if (dimsStr.startsWith('(') && dimsStr.endsWith(')')) dimsStr = dimsStr.slice(1, -1);
              dims = dimsStr.split(',').map(d => d.trim().replace(/^['"]|['"]$/g, ''));
            }

            const map = new Map();
            streams[src].data.forEach(row => {
              const key = dims.includes('__all__') ? '__all__' : dims.map(d => String(row[d] !== undefined ? row[d] : '')).join('||');
              if (!map.has(key)) map.set(key, []);
              map.get(key).push(row);
            });

            streams[target] = { data: streams[src].data, groups: map, groupDims: dims, isGrouped: true };
            lastStream = target;
            continue;
          }

          // FOREACH ... GENERATE
          const foreachMatch = expr.match(/^foreach\s+([a-zA-Z0-9_]+)\s+generate\s+([\s\S]+)$/i);
          if (foreachMatch) {
            const src = foreachMatch[1].trim(), gen = foreachMatch[2].trim();
            if (!streams[src]) throw new Error(`Stream "${src}" not defined.`);
            const projected = this.evalForeach(streams[src], gen);
            streams[target] = { data: projected, isGrouped: false };
            lastStream = target;
            continue;
          }

          // ORDER
          const orderMatch = expr.match(/^order\s+([a-zA-Z0-9_]+)\s+by\s+([\s\S]+)$/i);
          if (orderMatch) {
            const src = orderMatch[1].trim(), ord = orderMatch[2].trim();
            if (!streams[src]) throw new Error(`Stream "${src}" not defined.`);
            streams[target] = { data: this.evalOrder(streams[src].data, ord), isGrouped: streams[src].isGrouped };
            lastStream = target;
            continue;
          }

          // LIMIT
          const limitMatch = expr.match(/^limit\s+([a-zA-Z0-9_]+)\s+(\d+)$/i);
          if (limitMatch) {
            const src = limitMatch[1].trim(), lim = parseInt(limitMatch[2], 10);
            if (!streams[src]) throw new Error(`Stream "${src}" not defined.`);
            streams[target] = { data: streams[src].data.slice(0, lim), isGrouped: streams[src].isGrouped };
            lastStream = target;
            continue;
          }

          throw new Error(`Unrecognized SAQL clause in statement: "${stmt}".`);
        }

        const out = streams[lastStream] || { data: [] };
        return {
          data: out.data,
          dataset: detectedDataset,
          executionTimeMs: Math.max(1, Math.round(performance.now() - startTime))
        };
      }

      evalCondition(row, cond) {
        if (/\s+or\s+/i.test(cond)) return cond.split(/\s+or\s+/i).some(c => this.evalCondition(row, c.trim()));
        if (/\s+and\s+/i.test(cond)) return cond.split(/\s+and\s+/i).every(c => this.evalCondition(row, c.trim()));

        const inMatch = cond.match(/^['"]?([a-zA-Z0-9_]+)['"]?\s+in\s+\[([\s\S]*)\]$/i);
        if (inMatch) {
          const field = inMatch[1];
          const list = inMatch[2].split(',').map(s => s.trim().replace(/^['"]|['"]$/g, ''));
          return list.includes(String(row[field] !== undefined ? row[field] : ''));
        }

        const opMatch = cond.match(/^['"]?([a-zA-Z0-9_]+)['"]?\s*(==|!=|>=|<=|>|<)\s*(.+)$/i);
        if (!opMatch) return true;
        const field = opMatch[1], op = opMatch[2];
        let val = opMatch[3].trim().replace(/^['"]|['"]$/g, '');
        if (val === 'true') val = true;
        if (val === 'false') val = false;

        const rowVal = row[field];
        if (rowVal === undefined) return false;
        const numVal = Number(val);
        const isNum = !isNaN(numVal) && typeof rowVal === 'number';

        switch (op) {
          case '==': return isNum ? rowVal === numVal : String(rowVal).toLowerCase() === String(val).toLowerCase();
          case '!=': return isNum ? rowVal !== numVal : String(rowVal).toLowerCase() !== String(val).toLowerCase();
          case '>': return isNum ? rowVal > numVal : rowVal > val;
          case '<': return isNum ? rowVal < numVal : rowVal < val;
          case '>=': return isNum ? rowVal >= numVal : rowVal >= val;
          case '<=': return isNum ? rowVal <= numVal : rowVal <= val;
          default: return false;
        }
      }

      evalForeach(st, gen) {
        const rawProj = gen.split(/,(?![^(]*\))/).map(p => p.trim());
        const projs = rawProj.map(p => {
          const asM = p.match(/^([\s\S]+?)\s+as\s+['"]?([a-zA-Z0-9_]+)['"]?$/i);
          return asM ? { expr: asM[1].trim(), alias: asM[2].trim() } : { expr: p, alias: p.replace(/^['"]|['"]$/g, '') };
        });

        if (st.isCogrouped) {
          const res = [];
          st.groups.forEach(pair => {
            const r = {};
            projs.forEach(p => r[p.alias] = this.evalCogroupExpr(p.expr, pair, st.s1Name, st.s2Name));
            res.push(r);
          });
          return res;
        }

        if (st.isGrouped && st.groups) {
          const res = [];
          st.groups.forEach(rows => {
            const r = {};
            projs.forEach(p => r[p.alias] = this.evalExpr(p.expr, rows));
            res.push(r);
          });
          return res;
        }

        return st.data.map(row => {
          const r = {};
          projs.forEach(p => r[p.alias] = this.evalExpr(p.expr, [row]));
          return r;
        });
      }

      evalCogroupExpr(expr, pair, s1, s2) {
        let clean = expr.trim();
        const coal = clean.match(/^coalesce\s*\((.*),\s*(.+)\)$/i);
        if (coal) {
          const v = this.evalCogroupExpr(coal[1], pair, s1, s2);
          const fb = coal[2].replace(/^['"]|['"]$/g, '');
          return (v !== null && v !== undefined && v !== 0 && !isNaN(v)) ? v : (!isNaN(Number(fb)) ? Number(fb) : fb);
        }
        const cnt = clean.match(/^count\s*\(([a-zA-Z0-9_]*)\)$/i);
        if (cnt) {
          const target = cnt[1] ? cnt[1].trim() : s1;
          return (pair[target] || []).length;
        }
        const agg = clean.match(/^(sum|avg|min|max)\s*\(([a-zA-Z0-9_]+)\['([a-zA-Z0-9_]+)'\]\)$/i);
        if (agg) {
          const fn = agg[1].toLowerCase(), stName = agg[2], col = agg[3];
          return this.computeAgg(fn, col, pair[stName] || []);
        }
        const fld = clean.match(/^([a-zA-Z0-9_]+)\['([a-zA-Z0-9_]+)'\]$/);
        if (fld) {
          const rows = pair[fld[1]] || [];
          return rows.length > 0 && rows[0][fld[2]] !== undefined ? rows[0][fld[2]] : null;
        }
        return clean.replace(/^['"]|['"]$/g, '');
      }

      evalExpr(expr, rows) {
        const c = expr.trim();
        if (/^count\(\s*\)$/i.test(c)) return rows.length;
        const agg = c.match(/^(sum|avg|min|max|unique)\s*\(\s*['"]?([a-zA-Z0-9_]+)['"]?\s*\)$/i);
        if (agg) return this.computeAgg(agg[1].toLowerCase(), agg[2], rows);

        if (c.includes('+') || c.includes('-') || c.includes('*') || c.includes('/')) {
          let s = c.replace(/count\(\s*\)/gi, rows.length);
          s = s.replace(/(sum|avg|min|max|unique)\s*\(\s*['"]?([a-zA-Z0-9_]+)['"]?\s*\)/gi, (m, fn, col) => this.computeAgg(fn.toLowerCase(), col, rows));
          if (rows.length === 1) {
            s = s.replace(/['"]([a-zA-Z0-9_]+)['"]/g, (m, col) => rows[0][col] !== undefined ? rows[0][col] : 0);
          }
          try {
            if (/^[0-9\.\s\+\-\*\/\(\)]+$/.test(s)) return Math.round(Function(`'use strict'; return (${s})`)() * 100) / 100;
          } catch(e) {}
        }
        const bare = c.replace(/^['"]|['"]$/g, '');
        if (rows.length > 0 && rows[0][bare] !== undefined) return rows[0][bare];
        return !isNaN(Number(c)) ? Number(c) : c;
      }

      computeAgg(fn, col, rows) {
        if (fn === 'unique') {
          const set = new Set(rows.map(r => r[col]).filter(v => v !== undefined && v !== null));
          return set.size;
        }
        const vals = rows.map(r => Number(r[col]) || 0);
        if (vals.length === 0) return 0;
        if (fn === 'sum') return vals.reduce((a, b) => a + b, 0);
        if (fn === 'avg') return Math.round((vals.reduce((a, b) => a + b, 0) / vals.length) * 100) / 100;
        if (fn === 'min') return Math.min(...vals);
        if (fn === 'max') return Math.max(...vals);
        return 0;
      }

      evalOrder(data, ord) {
        const parts = ord.split(',').map(p => p.trim());
        const crits = parts.map(p => {
          const m = p.match(/^['"]?([a-zA-Z0-9_]+)['"]?\s*(asc|desc)?$/i);
          return m ? { field: m[1], dir: (m[2] || 'asc').toLowerCase() } : { field: p.replace(/^['"]|['"]$/g, ''), dir: 'asc' };
        });
        return [...data].sort((a, b) => {
          for (const crit of crits) {
            const va = a[crit.field], vb = b[crit.field];
            if (va === vb) continue;
            let c = (typeof va === 'number' && typeof vb === 'number') ? va - vb : String(va || '').localeCompare(String(vb || ''));
            return crit.dir === 'desc' ? -c : c;
          }
          return 0;
        });
      }
    }

    const engine = new SAQLEngine(DATASETS);
  </script>

  <script>
    let currentRecords = [];
    let currentSortColumn = null;
    let currentSortDir = 'asc';

    // DOM Elements
    const saqlEditorInput = document.getElementById('saqlEditorInput');
    const lineNumbers = document.getElementById('lineNumbers');
    const saqlStatusText = document.getElementById('saqlStatusText');
    const activeDatasetBadge = document.getElementById('activeDatasetBadge');
    const recordCountHeader = document.getElementById('recordCountHeader');
    const engineLatency = document.getElementById('engineLatency');
    const outputTableSummary = document.getElementById('outputTableSummary');
    const resultsTableHead = document.getElementById('resultsTableHead');
    const resultsTableBody = document.getElementById('resultsTableBody');

    // Synchronize Line Numbers
    function syncLineNumbers() {
      const lineCount = (saqlEditorInput.value.match(/\n/g) || []).length + 1;
      let numbersStr = '';
      for (let i = 1; i <= lineCount; i++) {
        numbersStr += i + '<br/>';
      }
      lineNumbers.innerHTML = numbersStr;
    }

    saqlEditorInput.addEventListener('scroll', () => {
      lineNumbers.scrollTop = saqlEditorInput.scrollTop;
    });

    saqlEditorInput.addEventListener('input', () => {
      syncLineNumbers();
    });

    // Native Tab indent handling (2 spaces) & Run Shortcut
    saqlEditorInput.addEventListener('keydown', (e) => {
      if (e.key === 'Tab') {
        e.preventDefault();
        const start = saqlEditorInput.selectionStart;
        const end = saqlEditorInput.selectionEnd;
        const val = saqlEditorInput.value;

        saqlEditorInput.value = val.substring(0, start) + '  ' + val.substring(end);
        saqlEditorInput.selectionStart = saqlEditorInput.selectionEnd = start + 2;
        syncLineNumbers();
      }

      if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') {
        e.preventDefault();
        executeCurrentSAQL();
      }
    });

    // Clean Tabular Data Formatting
    function formatCell(val, colName = '') {
      if (val === null || val === undefined) return '<span class="text-slate-300">-</span>';
      if (typeof val === 'number') {
        const cLower = colName.toLowerCase();
        if (cLower.includes('revenue') || cLower.includes('amount') || cLower.includes('quota') || cLower.includes('price') || cLower.includes('pipe')) {
          return '$' + val.toLocaleString();
        }
        if (cLower.includes('pct') || cLower.includes('attainment') || cLower.includes('probability')) {
          return val.toFixed(1) + '%';
        }
        return val.toLocaleString();
      }
      if (typeof val === 'boolean') {
        return val 
          ? '<span class="text-emerald-700 bg-emerald-50 border border-emerald-200 px-1.5 py-0.5 rounded text-[11px] font-bold">true</span>' 
          : '<span class="text-slate-600 bg-slate-100 border border-slate-200 px-1.5 py-0.5 rounded text-[11px]">false</span>';
      }
      return String(val);
    }

    // Render Tabular Output
    function renderTabularOutput(records) {
      if (!records || records.length === 0) {
        resultsTableHead.innerHTML = `<tr><th class="p-3 text-slate-400 font-normal">No Records</th></tr>`;
        resultsTableBody.innerHTML = `<tr><td class="p-8 text-center text-slate-400 font-sans">0 rows returned.</td></tr>`;
        outputTableSummary.innerText = `0 rows returned`;
        recordCountHeader.innerText = `0 rows returned`;
        return;
      }

      const columns = Object.keys(records[0]);
      outputTableSummary.innerText = `${records.length} rows returned`;
      recordCountHeader.innerText = `${records.length} rows returned`;

      // Header with click-to-sort
      resultsTableHead.innerHTML = `
        <tr>
          <th class="p-2.5 w-10 text-slate-400 font-mono text-[11px] border-b border-slate-200 bg-slate-100/90 text-center">#</th>
          ${columns.map(col => {
            const isSorted = currentSortColumn === col;
            const sortIcon = isSorted 
              ? (currentSortDir === 'asc' ? '<i class="fa-solid fa-arrow-up-short-wide text-crma-brand ml-1"></i>' : '<i class="fa-solid fa-arrow-down-wide-short text-crma-brand ml-1"></i>')
              : '<i class="fa-solid fa-sort text-slate-300 hover:text-slate-500 ml-1 text-[10px]"></i>';
            return `
              <th class="p-2.5 font-semibold text-slate-700 border-b border-slate-200 cursor-pointer hover:bg-slate-200/70 transition select-none" onclick="handleSort('${col}')">
                <div class="flex items-center justify-between">
                  <span>${col}</span>
                  ${sortIcon}
                </div>
              </th>
            `;
          }).join('')}
        </tr>
      `;

      // Body rows
      resultsTableBody.innerHTML = records.map((row, idx) => `
        <tr class="${idx % 2 === 0 ? 'bg-white' : 'bg-slate-50/60'} hover:bg-blue-50/60 transition">
          <td class="p-2.5 text-center text-slate-400 font-mono text-[11px] border-b border-slate-100">${idx + 1}</td>
          ${columns.map(col => `
            <td class="p-2.5 border-b border-slate-100 text-slate-800">${formatCell(row[col], col)}</td>
          `).join('')}
        </tr>
      `).join('');
    }

    // Client-side Column Sort
    window.handleSort = function(col) {
      if (currentSortColumn === col) {
        currentSortDir = currentSortDir === 'asc' ? 'desc' : 'asc';
      } else {
        currentSortColumn = col;
        currentSortDir = 'asc';
      }

      currentRecords.sort((a, b) => {
        const va = a[col], vb = b[col];
        if (va === vb) return 0;
        let diff = (typeof va === 'number' && typeof vb === 'number') 
          ? va - vb 
          : String(va || '').localeCompare(String(vb || ''));
        return currentSortDir === 'desc' ? -diff : diff;
      });

      renderTabularOutput(currentRecords);
    };

    // Execute Current SAQL
    function executeCurrentSAQL() {
      const code = saqlEditorInput.value;
      try {
        const result = engine.execute(code);
        currentRecords = result.data;
        currentSortColumn = null;

        activeDatasetBadge.innerText = result.dataset;
        engineLatency.innerText = `Latency: ${result.executionTimeMs}ms`;

        saqlStatusText.innerHTML = `
          <i class="fa-solid fa-circle-check text-emerald-400 text-xs"></i>
          <span class="text-emerald-300 font-medium">Executed successfully (${result.executionTimeMs}ms)</span>
        `;

        renderTabularOutput(currentRecords);
      } catch (err) {
        saqlStatusText.innerHTML = `
          <i class="fa-solid fa-triangle-exclamation text-rose-400 text-xs"></i>
          <span class="text-rose-300 font-medium">${escapeHtml(err.message)}</span>
        `;
      }
    }

    function escapeHtml(str) {
      return String(str).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
    }

    function showToast(msg, icon = 'fa-circle-check') {
      const toast = document.getElementById('toastNotification');
      document.getElementById('toastMessage').innerText = msg;
      document.getElementById('toastIcon').className = `fa-solid ${icon} text-emerald-400`;
      toast.classList.remove('translate-y-20', 'opacity-0');
      setTimeout(() => toast.classList.add('translate-y-20', 'opacity-0'), 2500);
    }
  </script>

  <script>
    // Run & Format Buttons
    document.getElementById('runSaqlBtn').addEventListener('click', executeCurrentSAQL);

    document.getElementById('formatSaqlBtn').addEventListener('click', () => {
      const c = saqlEditorInput.value;
      const stmts = c.split(';').map(s => s.trim()).filter(s => s.length > 0);
      saqlEditorInput.value = stmts.map(s => s + ';').join('\n');
      syncLineNumbers();
      showToast("Query formatted");
    });

    document.getElementById('copySaqlBtn').addEventListener('click', () => {
      navigator.clipboard.writeText(saqlEditorInput.value).then(() => showToast("SAQL copied to clipboard"));
    });

    // Templates Picker
    document.getElementById('templateSelector').addEventListener('change', (e) => {
      const key = e.target.value;
      if (TEMPLATES[key]) {
        saqlEditorInput.value = TEMPLATES[key];
        syncLineNumbers();
        executeCurrentSAQL();
      }
    });

    // Export CSV
    document.getElementById('exportCsvBtn').addEventListener('click', () => {
      if (!currentRecords || currentRecords.length === 0) {
        showToast("No data to export", "fa-circle-exclamation");
        return;
      }
      const headers = Object.keys(currentRecords[0]);
      const csv = [headers.join(',')].concat(currentRecords.map(r => headers.map(h => `"${r[h] !== undefined ? r[h] : ''}"`).join(','))).join('\n');
      const blob = new Blob([csv], { type: 'text/csv' });
      const a = document.createElement('a');
      a.href = URL.createObjectURL(blob);
      a.download = `saql_results_${Date.now()}.csv`;
      a.click();
      showToast("CSV exported successfully");
    });

    // Shareable URL Link (Encodes SAQL in hash for sharing across any browser)
    document.getElementById('shareLinkBtn').addEventListener('click', () => {
      const stateObj = { q: saqlEditorInput.value };
      const encoded = btoa(encodeURIComponent(JSON.stringify(stateObj)));
      const shareUrl = window.location.origin + window.location.pathname + '#saql=' + encoded;
      
      navigator.clipboard.writeText(shareUrl).then(() => {
        showToast("Sharable link copied to clipboard!");
      }).catch(() => {
        showToast("URL updated in address bar!");
      });
      window.location.hash = 'saql=' + encoded;
    });

    function loadStateFromURL() {
      if (window.location.hash && window.location.hash.startsWith('#saql=')) {
        try {
          const rawHash = window.location.hash.replace('#saql=', '');
          const decoded = decodeURIComponent(atob(rawHash));
          const parsed = JSON.parse(decoded);
          if (parsed.q) {
            saqlEditorInput.value = parsed.q;
            return true;
          }
        } catch(e) {
          console.warn("Could not load query from URL:", e);
        }
      }
      return false;
    }

    // Dataset Schema Modal
    let activeModalDs = 'opportunity';
    function renderSchemaModal(ds) {
      activeModalDs = ds;
      const data = engine.getDataset(ds);
      const schema = SCHEMAS[ds];

      document.querySelectorAll('.dataset-tab').forEach(t => {
        const isMatch = t.dataset.ds === ds;
        t.className = `dataset-tab text-xs pb-2 ${isMatch ? 'font-semibold border-b-2 border-crma-brand text-crma-brand' : 'font-medium text-slate-500 hover:text-slate-800'}`;
      });

      const grid = document.getElementById('schemaGridContent');
      grid.innerHTML = Object.keys(schema).map(col => `
        <div class="p-2.5 rounded bg-slate-50 border border-slate-200">
          <span class="font-bold text-slate-800 block truncate">${col}</span>
          <span class="text-[10px] text-slate-500">${schema[col]}</span>
        </div>
      `).join('');

      const tbl = document.getElementById('schemaSampleTable');
      const cols = Object.keys(data[0]);
      tbl.innerHTML = `
        <thead><tr class="bg-slate-50 border-b border-slate-200">${cols.map(c => `<th class="p-2 font-semibold text-slate-600">${c}</th>`).join('')}</tr></thead>
        <tbody>${data.slice(0, 4).map(r => `<tr>${cols.map(c => `<td class="p-2 border-b border-slate-100 font-mono text-[11px]">${r[c]}</td>`).join('')}</tr>`).join('')}</tbody>
      `;
    }

    document.getElementById('openSchemaBtn').addEventListener('click', () => {
      document.getElementById('schemaModal').classList.remove('hidden');
      renderSchemaModal(activeModalDs);
    });

    document.getElementById('closeSchemaModal').addEventListener('click', () => {
      document.getElementById('schemaModal').classList.add('hidden');
    });

    document.querySelectorAll('.dataset-tab').forEach(t => {
      t.addEventListener('click', () => renderSchemaModal(t.dataset.ds));
    });

    document.getElementById('insertDatasetLoadBtn').addEventListener('click', () => {
      saqlEditorInput.value = `q = load "${activeModalDs}";\nq = limit q 10;`;
      syncLineNumbers();
      document.getElementById('schemaModal').classList.add('hidden');
      executeCurrentSAQL();
    });

    // App Initialization
    window.addEventListener('DOMContentLoaded', () => {
      const loaded = loadStateFromURL();
      if (!loaded) {
        saqlEditorInput.value = TEMPLATES.pipeline_stage;
      }
      syncLineNumbers();
      executeCurrentSAQL();
    });
  </script>
</body>
</html>
