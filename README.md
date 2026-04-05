<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Iceland 2026 Tax Calculator</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 700px; margin: 2rem auto; }
    label, input { font-size: 1rem; }
    input { padding: 0.3rem; width: 200px; box-sizing: border-box;}
    .row { margin: 0.7rem 0; }
    table { border-collapse: collapse; width: 100%; margin-top: 1em; background: #fff; }
    th, td { border: 1px solid #ccc; padding: 0.7em; text-align: right; }
    th:first-child, td:first-child { text-align: left; }
    tfoot tr { background: #f8f8f8; }
    .explain { font-size: 0.96em; color: #555; margin-top: 1em; }
    #error { color: red; margin-left: 1em; font-size: 0.96em; }
    @media (max-width: 700px) {
      body { max-width: 100vw; margin: 1rem; font-size: 0.98em; }
      table { font-size: 0.97em; }
      input { width: 100%; }
    }
    canvas { display: block; margin: 1.5em 0 1em 0; max-width: 100%; border: none; }
  </style>
</head>
<body>
  <h1>Iceland 2026 Tax Comparison</h1>

  <form id="salaryForm" autocomplete="off" role="form" class="row">
    <label for="salary">Monthly gross salary (ISK): </label>
    <input id="salary" name="salary" type="number" min="0" step="1000" value="900000" required>
    <button type="submit">Calculate</button>
    <span id="error" aria-live="polite"></span>
  </form>

  <div id="results"></div>
  <div class="explain">
    <b>Effective rate:</b> Tax divided by gross salary. <br>
    <b>Credit:</b> Monthly personal tax-free amount deducted from payable tax.
  </div>
  <canvas id="chart" width="600" height="180"></canvas>
  <div class="explain">
    The chart shows net pay vs. gross salary for both systems.
  </div>

  <script>
    // --- TAX CALCS ---
    function calcCurrentSystem(gross) {
      const credit = 72492; // monthly credit
      let tax = 0;
      const b1 = 498000;
      const b2 = 1400000;
      if (gross <= b1) {
        tax = gross * 0.3149;
      } else if (gross <= b2) {
        tax = b1 * 0.3149 + (gross - b1) * 0.3799;
      } else {
        tax = b1 * 0.3149 + (b2 - b1) * 0.3799 + (gross - b2) * 0.4629;
      }
      tax = Math.max(0, tax - credit);
      const net = gross - tax;
      const effRate = gross > 0 ? (tax / gross) * 100 : 0;
      return { tax, net, effRate };
    }

    function calcFlatSystem(gross) {
      const credit = 100000; // monthly credit
      let tax = gross * 0.35;
      tax = Math.max(0, tax - credit);
      const net = gross - tax;
      const effRate = gross > 0 ? (tax / gross) * 100 : 0;
      return { tax, net, effRate };
    }

    // --- FORMATTING ---
    function formatISK(x) { return x.toLocaleString('is-IS', { maximumFractionDigits: 0 }); }
    function formatPct(x) { return x.toFixed(1) + ' %'; }

    // --- MAIN CALCULATE AND RENDER ---
    function calculate() {
      const salaryInput = document.getElementById('salary');
      const errorEl = document.getElementById('error');
      const el = document.getElementById('results');
      const gross = Number(salaryInput.value) || 0;
      errorEl.textContent = "";

      if (gross < 0) {
        errorEl.textContent = "Salary cannot be negative.";
        el.innerHTML = "";
        clearChart();
        return;
      }

      const cur = calcCurrentSystem(gross);
      const flat = calcFlatSystem(gross);
      const diffTax = flat.tax - cur.tax;
      const diffNet = flat.net - cur.net;

      el.innerHTML = `
        <table>
          <thead>
            <tr>
              <th>System</th>
              <th>Tax</th>
              <th>Effective rate</th>
              <th>Net pay</th>
              <th>Credit</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <td><b>Current progressive</b></td>
              <td>ISK ${formatISK(cur.tax)}</td>
              <td>${formatPct(cur.effRate)}</td>
              <td>ISK ${formatISK(cur.net)}</td>
              <td>ISK 72,492</td>
            </tr>
            <tr>
              <td><b>Flat 35% (+ 100,000 credit)</b></td>
              <td>ISK ${formatISK(flat.tax)}</td>
              <td>${formatPct(flat.effRate)}</td>
              <td>ISK ${formatISK(flat.net)}</td>
              <td>ISK 100,000</td>
            </tr>
          </tbody>
          <tfoot>
            <tr>
              <td><b>Difference (flat − current)</b></td>
              <td>ISK ${formatISK(diffTax)}</td>
              <td></td>
              <td>ISK ${formatISK(diffNet)}</td>
              <td></td>
            </tr>
          </tfoot>
        </table>
      `;

      drawChart();
    }

    // --- CHARTING ---
    function drawChart() {
      const canvas = document.getElementById('chart');
      if (!canvas.getContext) return;
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      // Axes
      ctx.beginPath();
      ctx.moveTo(50, 10); ctx.lineTo(50, 170); ctx.lineTo(580, 170);
      ctx.strokeStyle = "#bbb";
      ctx.lineWidth = 1;
      ctx.stroke();

      // Salary scale (x-axis)
      const maxGross = 2000000, step = 20000;
      ctx.font = "12px system-ui";
      ctx.fillStyle = "#444";
      ctx.fillText("Net pay (ISK)", 2, 20);
      ctx.fillText("Gross salary", 490, 179);
      // Y ticks
      for (let i = 0; i <= 2; ++i) {
        let income = (i / 2) * maxGross;
        let y = 170 - (income / maxGross) * 160;
        ctx.fillText(formatISK(income), 1, y + 7);
        ctx.beginPath();
        ctx.moveTo(48, y); ctx.lineTo(52, y); ctx.strokeStyle = "#ccc"; ctx.stroke();
      }
      // X ticks
      for (let i = 0; i <= 4; ++i) {
        let gross = (i / 4) * maxGross;
        let x = 50 + (gross/maxGross)*530;
        ctx.fillText(formatISK(gross), x - 18, 190);
        ctx.beginPath();
        ctx.moveTo(x, 168); ctx.lineTo(x, 172); ctx.strokeStyle = "#ccc"; ctx.stroke();
      }

      // Draw curves: progressive (blue) and flat (green)
      ctx.beginPath();
      for (let g=0; g<=maxGross; g+=step) {
        const np = calcCurrentSystem(g).net;
        const y = 170 - (np / maxGross) * 160;
        const x = 50 + (g/maxGross)*530;
        (g==0) ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
      }
      ctx.strokeStyle = "#0074d9";
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.fillStyle = "#0074d9"; ctx.fillText("Current", 110, 40);

      ctx.beginPath();
      for (let g=0; g<=maxGross; g+=step) {
        const np = calcFlatSystem(g).net;
        const y = 170 - (np / maxGross) * 160;
        const x = 50 + (g/maxGross)*530;
        (g==0) ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
      }
      ctx.strokeStyle = "#2ecc40";
      ctx.lineWidth = 2;
      ctx.stroke();
      ctx.fillStyle = "#2ecc40"; ctx.fillText("Flat", 180, 80);
    }

    function clearChart() {
      const canvas = document.getElementById('chart');
      if (canvas && canvas.getContext) {
        canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
      }
    }

    // --- URL PARAM ---

    function getSalaryFromURL() {
      const params = new URLSearchParams(window.location.search);
      const salary = params.get('salary');
      if (salary && !isNaN(salary)) {
        document.getElementById('salary').value = salary;
      }
    }

    // --- EVENT BINDINGS ---

    // On salary change, recalc automatically
    document.addEventListener('DOMContentLoaded', function() {
      getSalaryFromURL();
      document.getElementById('salaryForm').addEventListener('submit', function(e) {
        e.preventDefault();
        calculate();
      });
      document.getElementById('salary').addEventListener('input', calculate);
      calculate();
    });
  </script>
</body>
</html>
