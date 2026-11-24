# MILLIONAIRE-BOT-
mkdir -p src

cat > package.json <<'EOF'
{
  "name": "ojaybot-pro",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host 0.0.0.0",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@vitejs/plugin-react": "^4.3.4",
    "vite": "^6.0.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  }
}
EOF

cat > index.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <meta name="theme-color" content="#080d19" />
  <title>OJAYBOT PRO | Deriv Trading</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.jsx"></script>
</body>
</html>
EOF

cat > vite.config.js <<'EOF'
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()]
});
EOF

cat > vercel.json <<'EOF'
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/" }
  ]
}
EOF

cat > .gitignore <<'EOF'
node_modules
dist
.env
.env.local
.DS_Store
EOF

cat > src/main.jsx <<'EOF'
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./style.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
EOF

cat > src/App.jsx <<'EOF'
import { useEffect, useMemo, useState } from "react";

const MARKETS = [
  ["R_10", "Volatility 10"],
  ["R_25", "Volatility 25"],
  ["R_50", "Volatility 50"],
  ["R_75", "Volatility 75"],
  ["R_100", "Volatility 100"],
  ["1HZ10V", "Volatility 10 (1s)"],
  ["1HZ25V", "Volatility 25 (1s)"],
  ["1HZ50V", "Volatility 50 (1s)"],
  ["1HZ75V", "Volatility 75 (1s)"],
  ["1HZ100V", "Volatility 100 (1s)"]
];

export default function App() {
  const [market, setMarket] = useState("R_100");
  const [ticks, setTicks] = useState([]);
  const [status, setStatus] = useState("Connecting to market...");
  const [stake, setStake] = useState("0.35");
  const [account, setAccount] = useState("Demo");
  const [strategy, setStrategy] = useState("Automatic");

  useEffect(() => {
    let socket;
    let reconnectTimer;
    let active = true;

    function connect() {
      if (!active) return;

      socket = new WebSocket(
        "wss://ws.derivws.com/websockets/v3?app_id=1089"
      );

      socket.onopen = () => {
        if (!active) return;
        setStatus("Connected — receiving ticks");
        socket.send(JSON.stringify({
          ticks: market,
          subscribe: 1
        }));
      };

      socket.onmessage = (event) => {
        const data = JSON.parse(event.data);

        if (data.error) {
          setStatus(data.error.message);
          return;
        }

        if (data.tick) {
          const quoteText = String(data.tick.quote);
          const digit = Number(quoteText.replace(".", "").slice(-1));

          setTicks(old => [
            ...old,
            {
              quote: quoteText,
              digit,
              time: data.tick.epoch
            }
          ].slice(-100));
        }
      };

      socket.onerror = () => setStatus("Market connection error");

      socket.onclose = () => {
        if (active) {
          setStatus("Disconnected — reconnecting");
          reconnectTimer = setTimeout(connect, 3000);
        }
      };
    }

    setTicks([]);
    setStatus("Connecting to market...");
    connect();

    return () => {
      active = false;
      clearTimeout(reconnectTimer);
      if (socket) socket.close();
    };
  }, [market]);

  const stats = useMemo(() => {
    const counts = Array(10).fill(0);
    ticks.forEach(t => counts[t.digit]++);

    return {
      counts,
      even: ticks.filter(t => t.digit % 2 === 0).length,
      odd: ticks.filter(t => t.digit % 2 !== 0).length,
      over: ticks.filter(t => t.digit > 4).length,
      under: ticks.filter(t => t.digit < 5).length
    };
  }, [ticks]);

  const percent = n =>
    ticks.length ? `${(n / ticks.length * 100).toFixed(1)}%` : "0%";

  return (
    <main className="app">
      <header className="topbar">
        <div>
          <div className="brand">OJAYBOT <span>PRO</span></div>
          <small>DERIV TRADING INTELLIGENCE</small>
        </div>
        <div className="status">
          <i /> {status}
        </div>
      </header>

      <section className="hero">
        <p className="eyebrow">YOUR TRADING WORKSPACE</p>
        <h1>Trade smarter.<br /><span>Stay in control.</span></h1>
        <p className="sub">
          Monitor Volatility Indices and analyse the latest 100 ticks.
        </p>
      </section>

      <section className="panel">
        <h2>Trading settings</h2>
        <div className="fields">
          <label>
            Account
            <select value={account} onChange={e => setAccount(e.target.value)}>
              <option>Demo</option>
              <option>Real</option>
            </select>
          </label>

          <label>
            Market
            <select value={market} onChange={e => setMarket(e.target.value)}>
              {MARKETS.map(([value, name]) =>
                <option key={value} value={value}>{name}</option>
              )}
            </select>
          </label>

          <label>
            Stake (USD)
            <input
              type="number"
              min="0.35"
              step="0.01"
              value={stake}
              onChange={e => setStake(e.target.value)}
            />
          </label>

          <label>
            Strategy
            <select value={strategy} onChange={e => setStrategy(e.target.value)}>
              <option>Automatic</option>
              <option>Over</option>
              <option>Under</option>
              <option>Even</option>
              <option>Odd</option>
            </select>
          </label>
        </div>

        <div className="notice">
          {account === "Real"
            ? "Real account selected. Trading is not enabled."
            : "Demo selected. This version provides public market analysis only."}
        </div>

        <button className="trade-button" disabled>
          DERIV ACCOUNT AUTHENTICATION NOT YET CONNECTED
        </button>
        <small className="muted">
          No trades are placed by this starter version.
        </small>
      </section>

      <section className="stats">
        <div className="panel stat">
          <small>RECEIVED TICKS</small>
          <strong>{ticks.length}/100</strong>
        </div>
        <div className="panel stat">
          <small>EVEN DIGITS</small>
          <strong>{percent(stats.even)}</strong>
        </div>
        <div className="panel stat">
          <small>ODD DIGITS</small>
          <strong>{percent(stats.odd)}</strong>
        </div>
      </section>

      <section className="panel">
        <div className="section-heading">
          <div>
            <h2>Digit analysis</h2>
            <small>Last {ticks.length} received ticks</small>
          </div>
          <span className="live-label">LIVE FEED</span>
        </div>

        <div className="digit-grid">
          {stats.counts.map((count, digit) => (
            <div className="digit" key={digit}>
              <b>{digit}</b>
              <strong>{count}</strong>
              <small>{percent(count)}</small>
              <div className="bar">
                <span style={{
                  height: `${ticks.length ? count / ticks.length * 100 : 0}%`
                }} />
              </div>
            </div>
          ))}
        </div>
      </section>

      <section className="stats two">
        <div className="panel stat">
          <small>OVER 4 (DIGITS 5–9)</small>
          <strong>{percent(stats.over)}</strong>
        </div>
        <div className="panel stat">
          <small>UNDER 5 (DIGITS 0–4)</small>
          <strong>{percent(stats.under)}</strong>
        </div>
      </section>

      <section className="panel">
        <h2>Latest ticks</h2>
        <div className="tick-list">
          {ticks.slice(-12).reverse().map((tick, index) => (
            <div className="tick-row" key={`${tick.time}-${index}`}>
              <span>{new Date(tick.time * 1000).toLocaleTimeString()}</span>
              <b>{tick.quote}</b>
              <strong>{tick.digit}</strong>
            </div>
          ))}
          {ticks.length === 0 &&
            <p className="muted">Waiting for market ticks...</p>}
        </div>
      </section>

      <footer>
        OJAYBOT PRO · Market analysis only · Trading involves risk.
        Historical digit frequency does not predict the next tick.
      </footer>
    </main>
  );
}
EOF

cat > src/style.css <<'EOF'
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: #080d19;
  color: #f4f7ff;
  font-family: Arial, sans-serif;
}

.app {
  width: min(100% - 28px, 1050px);
  margin: auto;
  padding-bottom: 40px;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 0;
  border-bottom: 1px solid #202b40;
  gap: 12px;
}

.brand {
  font-size: 23px;
  font-weight: 900;
  letter-spacing: 1px;
}

.brand span,
.hero h1 span {
  color: #38e0b0;
}

.topbar small,
.eyebrow,
.muted {
  color: #8d9bb3;
}

.topbar small {
  font-size: 10px;
  letter-spacing: 2px;
}

.status {
  color: #aebbd0;
  font-size: 12px;
}

.status i {
  display: inline-block;
  width: 8px;
  height: 8px;
  background: #38e0b0;
  border-radius: 50%;
  margin-right: 5px;
}

.hero {
  padding: 42px 0 28px;
}

.eyebrow {
  font-size: 11px;
  letter-spacing: 2px;
}

.hero h1 {
  font-size: clamp(38px, 8vw, 65px);
  line-height: 1.05;
  margin: 12px 0;
}

.sub {
  color: #9eabc0;
  line-height: 1.6;
}

.panel {
  background: #111a2b;
  border: 1px solid #26344d;
  border-radius: 15px;
  padding: 20px;
  margin: 16px 0;
}

h2 {
  font-size: 18px;
  margin: 0 0 18px;
}

.fields {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

label {
  display: flex;
  flex-direction: column;
  gap: 8px;
  color: #aebbd0;
  font-size: 12px;
}

input,
select {
  width: 100%;
  padding: 13px;
  border-radius: 9px;
  border: 1px solid #35445f;
  background: #0b1322;
  color: white;
  outline: none;
}

input:focus,
select:focus {
  border-color: #38e0b0;
}

.notice {
  background: #19283a;
  color: #b7c7dd;
  padding: 13px;
  margin: 18px 0;
  border-radius: 9px;
  font-size: 12px;
  line-height: 1.5;
}

.trade-button {
  width: 100%;
  border: 0;
  border-radius: 9px;
  padding: 15px;
  background: #26344d;
  color: #aebbd0;
  font-weight: bold;
  cursor: not-allowed;
  font-size: 12px;
}

.muted {
  display: block;
  font-size: 11px;
  line-height: 1.5;
  margin-top: 12px;
}

.stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.stat {
  margin: 0;
}

.stat small {
  display: block;
  color: #8d9bb3;
  font-size: 10px;
  margin-bottom: 10px;
}

.stat strong {
  font-size: clamp(20px, 4vw, 28px);
}

.section-heading {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.section-heading h2 {
  margin-bottom: 5px;
}

.section-heading small {
  color: #8d9bb3;
}

.live-label {
  color: #38e0b0;
  font-size: 11px;
  font-weight: bold;
}

.digit-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 9px;
  margin-top: 20px;
}

.digit {
  text-align: center;
  background: #0b1322;
  border: 1px solid #26344d;
  border-radius: 9px;
  padding: 12px 5px;
}

.digit b {
  display: block;
  font-size: 20px;
  color: #38e0b0;
}

.digit strong {
  display: block;
  margin: 7px 0 3px;
}

.digit small {
  color: #8d9bb3;
  font-size: 10px;
}

.bar {
  height: 35px;
  display: flex;
  align-items: end;
  justify-content: center;
  margin-top: 8px;
}

.bar span {
  width: 45%;
  min-height: 2px;
  background: #38e0b0;
  border-radius: 5px 5px 0 0;
}

.tick-row {
  display: grid;
  grid-template-columns: 1fr 1fr 35px;
  align-items: center;
  padding: 11px 0;
  border-bottom: 1px solid #26344d;
  font-size: 12px;
}

.tick-row span {
  color: #8d9bb3;
}

.tick-row strong {
  text-align: center;
  color: #38e0b0;
  font-size: 17px;
}

footer {
  text-align: center;
  color: #718099;
  font-size: 11px;
  line-height: 1.6;
  padding: 25px 0;
}

@media (max-width: 500px) {
  .fields {
    grid-template-columns: 1fr;
  }

  .stats {
    grid-template-columns: repeat(2, 1fr);
  }

  .topbar {
    align-items: flex-start;
    flex-direction: column;
  }

  .digit-grid {
    gap: 5px;
  }

  .digit {
    padding: 10px 2px;
  }
}
EOF

npm install
npm run build
