---
title: "SaaS savings calculator: what would your stack cost self-hosted?"
description: "Enter what you pay for software today and see what self-hosting would really cost, including your time."
last_checked: 2026-10-31
---

Self-hosting isn't free: you pay for a server and you spend some time on updates and backups. This calculator puts both next to what you pay now, so you can see whether it's worth it for **your** business. It runs entirely in your browser; nothing you type is sent anywhere.

The example rows use made-up bills. **Replace them with your own** from your invoices; a price we quote could be out of date or wrong for your plan.

<div id="calc" class="calc">
  <table>
    <thead>
      <tr><th>Paid tool you use now</th><th>Self-hosted replacement</th><th>You pay per month ($)</th><th>Replace?</th></tr>
    </thead>
    <tbody id="rows"></tbody>
  </table>
  <p><button type="button" id="add">+ Add a tool</button></p>

  <fieldset>
    <legend>Self-hosting costs</legend>
    <label>Server, per month ($) <input id="server" type="number" min="0" step="1" value="20"></label>
    <label>Off-site backup storage, per month ($) <input id="backup" type="number" min="0" step="1" value="3"></label>
    <label>Your time on upkeep, hours per month <input id="hours" type="number" min="0" step="0.5" value="1"></label>
    <label>What an hour of your time is worth ($) <input id="rate" type="number" min="0" step="5" value="40"></label>
  </fieldset>

  <div id="result" class="result" aria-live="polite"></div>
</div>

<script>
(function () {
  var examples = [
    ["Scheduling tool (e.g. Calendly)", "Cal.com", 36, true],
    ["Password manager for the team", "Vaultwarden", 20, true],
    ["Uptime monitoring service", "Uptime Kuma", 10, true],
    ["Web analytics service", "Matomo", 0, false],
    ["Extra cloud storage", "Nextcloud", 30, false]
  ];
  var rows = document.getElementById("rows");
  function money(n) { return "$" + Math.round(n).toLocaleString("en-US"); }
  function addRow(r) {
    r = r || ["", "", 0, true];
    var tr = document.createElement("tr");
    tr.innerHTML =
      '<td><input type="text" class="tool" placeholder="Paid tool" aria-label="Paid tool"></td>' +
      '<td><input type="text" class="alt" placeholder="Self-hosted replacement" aria-label="Self-hosted replacement"></td>' +
      '<td><input type="number" class="cost" min="0" step="1" placeholder="$ per month" aria-label="You pay per month in dollars"></td>' +
      '<td><label><input type="checkbox" class="use"> replace</label> <button type="button" class="del" aria-label="Remove row">✕</button></td>';
    tr.querySelector(".tool").value = r[0];
    tr.querySelector(".alt").value = r[1];
    tr.querySelector(".cost").value = r[2];
    tr.querySelector(".use").checked = r[3];
    tr.querySelector(".del").onclick = function () { tr.remove(); calc(); };
    rows.appendChild(tr);
  }
  function num(id) { var v = parseFloat(document.getElementById(id).value); return isNaN(v) ? 0 : v; }
  function calc() {
    var saas = 0, n = 0;
    rows.querySelectorAll("tr").forEach(function (tr) {
      if (tr.querySelector(".use").checked) {
        var c = parseFloat(tr.querySelector(".cost").value);
        if (!isNaN(c)) { saas += c; n++; }
      }
    });
    var cash = num("server") + num("backup");
    var time = num("hours") * num("rate");
    var cashSave = saas - cash, fullSave = saas - cash - time;
    var verdict = n === 0 ? "Tick at least one tool to replace."
      : fullSave > 0 ? "Self-hosting these looks worth it, even counting your time."
      : cashSave > 0 ? "You'd save cash, but not once your time is counted. Worth it only if you enjoy it or need the control."
      : "Not worth it on cost alone. Keep paying unless privacy or control matters more to you.";
    document.getElementById("result").innerHTML =
      "<p><strong>Now:</strong> " + money(saas) + "/month (" + money(saas * 12) + "/year) for " + n + " tool" + (n === 1 ? "" : "s") + ".</p>" +
      "<p><strong>Self-hosted:</strong> " + money(cash) + "/month in server and storage, plus " + money(time) + "/month of your time.</p>" +
      "<p><strong>Yearly savings:</strong> " + money(cashSave * 12) + " in cash, " + money(fullSave * 12) + " counting your time.</p>" +
      "<p>" + verdict + "</p>";
  }
  examples.forEach(addRow);
  document.getElementById("add").onclick = function () { addRow(); calc(); };
  document.getElementById("calc").addEventListener("input", calc);
  calc();
})();
</script>

<style>
.calc table { width: 100%; border-collapse: collapse; }
.calc td, .calc th { padding: 4px 6px; vertical-align: middle; }
.calc input[type=text], .calc input[type=number] { width: 100%; box-sizing: border-box; padding: 4px; }
.calc fieldset { margin: 16px 0; border: 1px solid #ddd; padding: 12px; }
.calc fieldset label { display: block; margin: 6px 0; }
.calc fieldset input[type=number] { width: 120px; margin-left: 8px; display: inline-block; }
.calc .result { background: #f4f8fa; border-left: 4px solid #16425b; padding: 8px 16px; }
@media screen and (max-width: 600px) {
  .calc thead { display: none; }
  .calc tr { display: block; border-bottom: 1px solid #ddd; padding: 6px 0; }
  .calc td { display: block; }
}
</style>

## How to read the result

- **Server cost:** $20/month is a 4 GB server at monthly-billed providers; see [Which VPS?](./09-which-vps.md) for current prices. One server runs all the apps above.
- **Your time:** about an hour a month covers updates and checking backups for a handful of apps once they're set up. The first setup takes longer: an afternoon with our guides.
- **Things a calculator can't price:** owning your data, no per-seat fees as you hire, and the risk that you're now the one responsible when something breaks. Set up [backups](./08-backups.md) and [monitoring](./02-uptime-kuma.md) before you rely on anything.

Ready to try? Start with [the 30-minute server setup](./01-the-7-dollar-server.md).
