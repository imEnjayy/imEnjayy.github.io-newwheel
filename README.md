import React, { useMemo, useState } from "react";
import { Upload, Users, Coins, Percent, BarChart3, FileDown, Search } from "lucide-react";
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, PieChart, Pie, Cell } from "recharts";
import Papa from "papaparse";

/**
 * Affiliate Metrics Dashboard — single-file React app
 *
 * What it does (client-side only):
 * - Upload two CSVs (Campaign summary + User-level values) exported from Stake affiliate dashboard
 * - Parses and reconciles numbers, computes helpful KPIs
 * - Filters by username to understand a single player's contribution
 * - Visualizes top contributors and user distribution
 * - Exports a quick CSV of computed metrics
 *
 * How to deploy on GitHub Pages:
 * 1) Create a repo, add this file as `index.jsx` (or `src/App.jsx` in a Vite/Next/CRA app).
 * 2) If using a plain single-file setup here, click "Download" in ChatGPT to save, then push to GitHub.
 * 3) Enable GitHub Pages: Settings → Pages → Deploy from branch → `main` / root (or `docs/`).
 * 4) If using Vite/CRA/Next, build and publish the output to Pages.
 */

export default function App() {
  const [campaignCSV, setCampaignCSV] = useState(null);
  const [usersCSV, setUsersCSV] = useState(null);
  const [campaign, setCampaign] = useState(null); // parsed single-row object
  const [users, setUsers] = useState([]); // parsed rows
  const [usernameQuery, setUsernameQuery] = useState("");
  const [manualCommissionRate, setManualCommissionRate] = useState(0.30); // default 30%
  const [observedUserCommission, setObservedUserCommission] = useState(""); // optional actual value user sees in dashboard for a single player

  const parseCSV = (file, onDone) => {
    Papa.parse(file, {
      header: true,
      dynamicTyping: true,
      skipEmptyLines: true,
      complete: (results) => onDone(results.data),
    });
  };

  const onCampaignUpload = (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    setCampaignCSV(file);
    parseCSV(file, (rows) => {
      // Expecting a single-row CSV
      const row = rows?.[0] || null;
      setCampaign(row);
    });
  };

  const onUsersUpload = (e) => {
    const file = e.target.files?.[0];
    if (!file) return;
    setUsersCSV(file);
    parseCSV(file, (rows) => {
      // Normalize keys we expect
      const normalized = rows.map((r) => ({
        campaign: r.campaign ?? r["campaign"],
        username: String(r.username ?? r["username"] ?? "").trim(),
        created_at: r["created_at (UTC)"] ?? r.created_at ?? r["created_at"],
        value_usd: r["value (USD)"] ?? r.value ?? r["value_usd"],
      }));
      setUsers(normalized);
    });
  };

  // Helper: safely number
  const num = (x) => {
    const n = typeof x === "string" ? Number(String(x).replace(/[$,%\s]/g, "")) : Number(x);
    return isFinite(n) ? n : 0;
  };

  // Aggregate from campaign summary
  const campaignAgg = useMemo(() => {
    if (!campaign) return null;
    const referred_users = num(campaign.referred_users);
    const first_time_depositors = num(campaign.first_time_depositors);
    const total_deposits = num(campaign.total_deposits);
    const commission_rate = String(campaign.commission_rate ?? "").includes("%")
      ? num(campaign.commission_rate) / 100
      : (campaign.commission_rate ? num(campaign.commission_rate) : manualCommissionRate);
    const overall_commission = num(campaign["overall_commission (USD)"] ?? campaign.overall_commission);
    const overall_available_commission = num(campaign["overall_available_commission (USD)"] ?? campaign.overall_available_commission);

    return {
      campaign_name: campaign.campaign_name ?? campaign.campaign,
      referred_users,
      first_time_depositors,
      total_deposits,
      commission_rate,
      overall_commission,
      overall_available_commission,
      campaign_hits: num(campaign.campaign_hits),
      created: campaign.campaign_created_date ?? campaign.created_at,
      offer_code: campaign.offer_code ?? campaign.campaign_code,
    };
  }, [campaign, manualCommissionRate]);

  // Aggregate from users file
  const userAgg = useMemo(() => {
    if (!users?.length) return null;
    const byUser = new Map();
    let totalValue = 0;
    let rowsWithValue = 0;

    for (const row of users) {
      const u = row.username || "";
      const v = num(row.value_usd);
      if (!byUser.has(u)) byUser.set(u, { username: u, entries: 0, total_value: 0 });
      const rec = byUser.get(u);
      rec.entries += 1;
      rec.total_value += v;
      totalValue += v;
      if (v) rowsWithValue += 1;
    }

    const usersWithValue = Array.from(byUser.values()).filter((r) => r.total_value > 0).length;

    return {
      byUser,
      totalUsers: byUser.size,
      totalValue,
      usersWithValue,
      rowsWithValue,
      topUsers: Array.from(byUser.values())
        .sort((a, b) => b.total_value - a.total_value)
        .slice(0, 10),
    };
  }, [users]);

  // KPIs derived by reconciling both files
  const kpis = useMemo(() => {
    if (!campaignAgg || !userAgg) return null;
    const conversion = campaignAgg.referred_users
      ? campaignAgg.first_time_depositors / campaignAgg.referred_users
      : 0;

    const valuePerUser = userAgg.totalUsers ? userAgg.totalValue / userAgg.totalUsers : 0;
    const valuePerDepositor = campaignAgg.first_time_depositors
      ? userAgg.totalValue / campaignAgg.first_time_depositors
      : 0;

    const commissionPerUser = userAgg.totalUsers ? campaignAgg.overall_commission / userAgg.totalUsers : 0;
    const commissionPerDepositor = campaignAgg.first_time_depositors
      ? campaignAgg.overall_commission / campaignAgg.first_time_depositors
      : 0;

    const effectiveCommissionRate = userAgg.totalValue
      ? campaignAgg.overall_commission / userAgg.totalValue
      : 0;

    return {
      conversion,
      valuePerUser,
      valuePerDepositor,
      commissionPerUser,
      commissionPerDepositor,
      effectiveCommissionRate,
    };
  }, [campaignAgg, userAgg]);

  // Single user lookup (e.g., "nur1988")
  const userDetail = useMemo(() => {
    if (!userAgg) return null;
    const key = usernameQuery.trim();
    if (!key) return null;
    const rec = userAgg.byUser.get(key);
    if (!rec) return { username: key, entries: 0, total_value: 0 };
    const estCommission = rec.total_value * (campaignAgg?.commission_rate ?? manualCommissionRate);
    const observed = num(observedUserCommission);
    return {
      ...rec,
      estCommission,
      observedCommission: observed || null,
      variance: observed ? observed - estCommission : null,
    };
  }, [usernameQuery, userAgg, campaignAgg, manualCommissionRate, observedUserCommission]);

  // Simple CSV export of headline KPIs
  const downloadCSV = () => {
    if (!campaignAgg || !userAgg || !kpis) return;
    const lines = [
      ["Metric", "Value"],
      ["Campaign Name", campaignAgg.campaign_name ?? ""],
      ["Offer Code", campaignAgg.offer_code ?? ""],
      ["Campaign Hits", campaignAgg.campaign_hits],
      ["Referred Users", campaignAgg.referred_users],
      ["First-time Depositors", campaignAgg.first_time_depositors],
      ["Total Deposits (count)", campaignAgg.total_deposits],
      ["Commission Rate", (campaignAgg.commission_rate * 100).toFixed(2) + "%"],
      ["Overall Commission (USD)", campaignAgg.overall_commission.toFixed(2)],
      ["Overall Available Commission (USD)", campaignAgg.overall_available_commission.toFixed(2)],
      ["Unique Users in User CSV", userAgg.totalUsers],
      ["Users With Value", userAgg.usersWithValue],
      ["Total Value (USD)", userAgg.totalValue.toFixed(2)],
      ["Conversion (to depositor)", (kpis.conversion * 100).toFixed(2) + "%"],
      ["Value per User", kpis.valuePerUser.toFixed(2)],
      ["Value per Depositor", kpis.valuePerDepositor.toFixed(2)],
      ["Commission per User", kpis.commissionPerUser.toFixed(2)],
      ["Commission per Depositor", kpis.commissionPerDepositor.toFixed(2)],
      ["Effective Commission vs Value", (kpis.effectiveCommissionRate * 100).toFixed(3) + "%"],
    ]
      .map((r) => r.join(","))
      .join("\n");

    const blob = new Blob([lines], { type: "text/csv;charset=utf-8;" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "affiliate_kpis.csv";
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  };

  const nice = (n) => new Intl.NumberFormat(undefined, { maximumFractionDigits: 2 }).format(n ?? 0);
  const pct = (n) => (n ? (n * 100).toFixed(2) + "%" : "0.00%");

  const TopBar = () => (
    <header className="w-full sticky top-0 z-10 bg-white/70 backdrop-blur border-b">
      <div className="max-w-6xl mx-auto flex items-center gap-3 px-4 py-3">
        <BarChart3 className="w-6 h-6" />
        <h1 className="text-xl font-semibold">Affiliate Metrics Dashboard</h1>
        <div className="ml-auto flex items-center gap-2 text-sm opacity-70">
          <span>Client-side • No sign-in</span>
        </div>
      </div>
    </header>
  );

  const Uploader = () => (
    <section className="max-w-6xl mx-auto px-4 py-6 grid grid-cols-1 md:grid-cols-2 gap-4">
      <div className="p-4 rounded-2xl border shadow-sm">
        <div className="flex items-center gap-2 mb-2">
          <Upload className="w-5 h-5" />
          <h2 className="font-medium">Upload Campaign Summary CSV</h2>
        </div>
        <input type="file" accept=".csv" onChange={onCampaignUpload} className="block w-full" />
        <p className="text-sm mt-2 text-gray-600">Expected headers: campaign_name, campaign_code, campaign_created_date, campaign_hits, referred_users, first_time_depositors, total_deposits, commission_rate, overall_commission (USD), overall_available_commission (USD), offer_code</p>
      </div>

      <div className="p-4 rounded-2xl border shadow-sm">
        <div className="flex items-center gap-2 mb-2">
          <Upload className="w-5 h-5" />
          <h2 className="font-medium">Upload User-Level CSV</h2>
        </div>
        <input type="file" accept=".csv" onChange={onUsersUpload} className="block w-full" />
        <p className="text-sm mt-2 text-gray-600">Expected headers: campaign, username, created_at (UTC), value (USD)</p>
      </div>

      <div className="p-4 rounded-2xl border shadow-sm md:col-span-2 flex flex-wrap items-center gap-4">
        <div>
          <label className="text-sm text-gray-600">Commission rate</label>
          <input
            type="number"
            step="0.01"
            min="0"
            max="1"
            value={manualCommissionRate}
            onChange={(e) => setManualCommissionRate(Number(e.target.value))}
            className="ml-2 border rounded-lg px-2 py-1 w-28"
          />
          <span className="ml-2 text-sm text-gray-500">(1 = 100%, default 0.30)</span>
        </div>
        <button onClick={downloadCSV} className="ml-auto px-3 py-2 rounded-xl border shadow-sm hover:shadow transition flex items-center gap-2">
          <FileDown className="w-4 h-4" /> Export KPIs CSV
        </button>
      </div>
    </section>
  );

  const KPICards = () => {
    if (!campaignAgg || !userAgg || !kpis) return null;
    const cards = [
      { icon: <Users className="w-5 h-5" />, label: "Referred Users", value: nice(campaignAgg.referred_users) },
      { icon: <Coins className="w-5 h-5" />, label: "Users With Value", value: nice(userAgg.usersWithValue) },
      { icon: <Percent className="w-5 h-5" />, label: "Conversion (to depositor)", value: pct(kpis.conversion) },
      { icon: <Coins className="w-5 h-5" />, label: "Total Value (USD)", value: "$" + nice(userAgg.totalValue) },
      { icon: <Coins className="w-5 h-5" />, label: "Overall Commission (USD)", value: "$" + nice(campaignAgg.overall_commission) },
      { icon: <Percent className="w-5 h-5" />, label: "Effective Commission vs Value", value: pct(kpis.effectiveCommissionRate) },
    ];

    return (
      <section className="max-w-6xl mx-auto px-4 grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 pb-6">
        {cards.map((c, i) => (
          <div key={i} className="p-4 rounded-2xl border shadow-sm">
            <div className="flex items-center gap-2 text-gray-600 mb-1">{c.icon}<span className="text-sm">{c.label}</span></div>
            <div className="text-2xl font-semibold">{c.value}</div>
          </div>
        ))}
      </section>
    );
  };

  const Charts = () => {
    if (!userAgg) return null;
    const top = userAgg.topUsers.map((u) => ({ name: u.username, value: Math.round(u.total_value * 100) / 100 }));
    const distribution = [
      { name: "With Value", value: userAgg.usersWithValue },
      { name: "No Value", value: Math.max(userAgg.totalUsers - userAgg.usersWithValue, 0) },
    ];

    return (
      <section className="max-w-6xl mx-auto px-4 grid grid-cols-1 lg:grid-cols-2 gap-6 pb-10">
        <div className="p-4 rounded-2xl border shadow-sm">
          <h3 className="font-medium mb-2">Top 10 Users by Value</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={top} margin={{ left: 10, right: 10 }}>
                <XAxis dataKey="name" hide={false} interval={0} angle={-30} textAnchor="end" height={60} />
                <YAxis />
                <Tooltip formatter={(v) => "$" + nice(v)} />
                <Bar dataKey="value" />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </div>

        <div className="p-4 rounded-2xl border shadow-sm">
          <h3 className="font-medium mb-2">Users With vs Without Value</h3>
          <div className="h-72">
            <ResponsiveContainer width="100%" height="100%">
              <PieChart>
                <Pie data={distribution} dataKey="value" nameKey="name" cx="50%" cy="50%" outerRadius={96} label>
                  {distribution.map((entry, index) => (
                    <Cell key={`cell-${index}`} />
                  ))}
                </Pie>
                <Tooltip />
              </PieChart>
            </ResponsiveContainer>
          </div>
        </div>
      </section>
    );
  };

  const UserInspector = () => (
    <section className="max-w-6xl mx-auto px-4 pb-12">
      <div className="p-4 rounded-2xl border shadow-sm">
        <div className="flex items-center gap-2 mb-3">
          <Search className="w-5 h-5" />
          <h3 className="font-medium">User Inspector</h3>
        </div>
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div>
            <label className="text-sm text-gray-600">Username</label>
            <input
              type="text"
              placeholder="e.g., nur1988"
              className="block w-full border rounded-xl px-3 py-2 mt-1"
              value={usernameQuery}
              onChange={(e) => setUsernameQuery(e.target.value)}
            />
          </div>
          <div>
            <label className="text-sm text-gray-600">Observed Commission for this user (optional, USD)</label>
            <input
              type="number"
              step="0.01"
              className="block w-full border rounded-xl px-3 py-2 mt-1"
              placeholder="If you see a different number in the dashboard, enter it here"
              value={observedUserCommission}
              onChange={(e) => setObservedUserCommission(e.target.value)}
            />
          </div>
        </div>

        {userDetail && (
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mt-6">
            <div className="p-4 rounded-xl border">
              <div className="text-sm text-gray-600">Entries</div>
              <div className="text-xl font-semibold">{nice(userDetail.entries)}</div>
            </div>
            <div className="p-4 rounded-xl border">
              <div className="text-sm text-gray-600">Total Value (USD)</div>
              <div className="text-xl font-semibold">${nice(userDetail.total_value)}</div>
            </div>
            <div className="p-4 rounded-xl border">
              <div className="text-sm text-gray-600">Est. Commission (@ {pct(campaignAgg?.commission_rate ?? manualCommissionRate)})</div>
              <div className="text-xl font-semibold">${nice(userDetail.estCommission)}</div>
            </div>
            {userDetail.observedCommission !== null && (
              <div className="p-4 rounded-xl border md:col-span-3">
                <div className="text-sm text-gray-600">Observed Commission (USD)</div>
                <div className="text-xl font-semibold">${nice(userDetail.observedCommission)}</div>
                <div className="text-sm mt-1">Variance vs estimate: <span className={userDetail.variance >= 0 ? "text-green-600" : "text-red-600"}>{userDetail.variance >= 0 ? "+" : ""}{nice(userDetail.variance)}</span></div>
                <p className="text-sm text-gray-600 mt-2">Note: Differences are expected because Stake applies adjustments (bonuses, rakeback, provider fees, promotions, risk) before finalizing commissionable revenue.</p>
              </div>
            )}
          </div>
        )}
      </div>
    </section>
  );

  return (
    <div className="min-h-screen bg-gradient-to-b from-white to-slate-50">
      <TopBar />
      <Uploader />
      {campaignAgg && userAgg && (
        <>
          <KPICards />
          <Charts />
          <UserInspector />
        </>
      )}

      {!campaignAgg && !userAgg && (
        <section className="max-w-3xl mx-auto p-6 text-center text-gray-600">
          <p className="mb-2">Upload your two CSV exports to get started.</p>
          <p className="text-sm">This tool runs entirely in your browser. No data leaves your device.</p>
        </section>
      )}
    </div>
  );
}
