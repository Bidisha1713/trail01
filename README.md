
from flask import Flask, request, jsonify, render_template_string
import sqlite3
from datetime import datetime

app = Flask(__name__)

#  DATABASE
conn = sqlite3.connect('iot_data.db', check_same_thread=False)
c = conn.cursor()
c.execute('''
CREATE TABLE IF NOT EXISTS sensor_data (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT,
    power REAL,
    co REAL,
    co2 REAL,
    ethanol REAL,
    nh3 REAL,
    toluene REAL,
    acetone REAL,
    bin_fill REAL
)
''')
conn.commit()

# USER DATA
users = {"Alice": 10, "Bob": 15, "Charlie": 5}

# HELPER
def calculate_carbon_footprint(power_w, co2_ppm):
    footprint_energy = power_w * 0.92 / 1000
    footprint_air = co2_ppm * 0.01
    return round(footprint_energy + footprint_air, 2)

# ROUTES
@app.route("/api/iot/readings", methods=["POST"])
def receive_readings():
    data = request.get_json()
    if not data:
        return jsonify({"status": "error", "message": "No JSON received"}), 400

    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    c.execute('''
        INSERT INTO sensor_data 
        (timestamp, power, co, co2, ethanol, nh3, toluene, acetone, bin_fill)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
    ''', (
        timestamp,
        data.get("Power_W",0),
        data.get("CO_ppm",0),
        data.get("CO2_ppm",0),
        data.get("Ethanol_ppm",0),
        data.get("NH3_ppm",0),
        data.get("Toluene_ppm",0),
        data.get("Acetone_ppm",0),
        data.get("bin_fill_percent",0)
    ))
    conn.commit()
    return jsonify({"status": "success"}), 200

@app.route("/api/iot/latest")
def latest_readings():
    c.execute("SELECT timestamp, power, co, co2, ethanol, nh3, toluene, acetone, bin_fill FROM sensor_data ORDER BY id DESC LIMIT 1")
    row = c.fetchone()
    if not row:
        # Return dummy data if no ESP32 readings yet
        return jsonify({
            "timestamp": datetime.now().strftime("%H:%M:%S"),
            "Power_W": 0,
            "CO_ppm": 0,
            "CO2_ppm": 400,
            "Ethanol_ppm": 0,
            "NH3_ppm": 0,
            "Toluene_ppm": 0,
            "Acetone_ppm": 0,
            "bin_fill_percent": 0,
            "carbon_footprint": 0
        })
    return jsonify({
        "timestamp": row[0],
        "Power_W": row[1],
        "CO_ppm": row[2],
        "CO2_ppm": row[3],
        "Ethanol_ppm": row[4],
        "NH3_ppm": row[5],
        "Toluene_ppm": row[6],
        "Acetone_ppm": row[7],
        "bin_fill_percent": row[8],
        "carbon_footprint": calculate_carbon_footprint(row[1], row[3])
    })

@app.route("/")
def dashboard():
    html = """
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>Smart Green Campus Dashboard</title>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
body {background-color:#f0f8f0; padding:20px;}
.card {margin-bottom:20px;}
</style>
</head>
<body>
<div class="container">
  <h1 class="mb-4 text-center">🌿 Smart Green Campus Dashboard</h1>

  <div class="row">
    <div class="col-md-6">
      <div class="card p-3">
        <h5>Energy Usage (W)</h5>
        <canvas id="energyChart"></canvas>
      </div>
    </div>
    <div class="col-md-6">
      <div class="card p-3">
        <h5>CO2 Levels (ppm)</h5>
        <canvas id="co2Chart"></canvas>
      </div>
    </div>
  </div>

  <div class="row">
    <div class="col-md-6">
      <div class="card p-3">
        <h5>Bin Fill (%)</h5>
        <canvas id="binChart"></canvas>
      </div>
    </div>
    <div class="col-md-6">
      <div class="card p-3">
        <h5>Carbon Footprint (kg CO2)</h5>
        <h2 id="carbonValue" class="text-center text-success">0</h2>
      </div>
    </div>
  </div>

  <div class="row">
    <div class="col-md-6">
      <div class="card p-3">
        <h5>Eco-Score Leaderboard</h5>
        <ul id="leaderboard"></ul>
      </div>
    </div>
    <div class="col-md-6">
      <div class="card p-3">
        <h5>AI Suggestions</h5>
        <div id="suggestions"></div>
      </div>
    </div>
  </div>
</div>

<script>
let energyChart, co2Chart, binChart;
let energyData = [0], co2Data = [400], binData = [0], labels = [new Date().toLocaleTimeString()];

function initCharts() {
  const ctx1 = document.getElementById('energyChart').getContext('2d');
  energyChart = new Chart(ctx1, {
    type:'line',
    data:{labels:labels,datasets:[{label:'Power W',data:energyData,borderColor:'blue',fill:false}]},
    options:{animation:false, responsive:true, maintainAspectRatio:false}
  });

  const ctx2 = document.getElementById('co2Chart').getContext('2d');
  co2Chart = new Chart(ctx2, {
    type:'line',
    data:{labels:labels,datasets:[{label:'CO2 ppm',data:co2Data,borderColor:'green',fill:false}]},
    options:{animation:false,responsive:true, maintainAspectRatio:false}
  });

  const ctx3 = document.getElementById('binChart').getContext('2d');
  binChart = new Chart(ctx3, {
    type:'line',
    data:{labels:labels,datasets:[{label:'Bin Fill %',data:binData,borderColor:'orange',fill:false}]},
    options:{animation:false,responsive:true, maintainAspectRatio:false}
  });
}

async function fetchData() {
  try {
    const res = await fetch('/api/iot/latest');
    const data = await res.json();
    if(!data || !data.timestamp) return;

    if(labels.length>50){labels.shift(); energyData.shift(); co2Data.shift(); binData.shift();}
    labels.push(new Date().toLocaleTimeString());
    energyData.push(data.Power_W || 0);
    co2Data.push(data.CO2_ppm || 0);
    binData.push(data.bin_fill_percent || 0);

    energyChart.update();
    co2Chart.update();
    binChart.update();

    document.getElementById('carbonValue').innerText = data.carbon_footprint || 0;

    // Suggestions
    let suggestions = [];
    if(data.Power_W>50) suggestions.push("⚡ High energy usage! Turn off unused appliances.");
    else suggestions.push("✅ Energy usage is good.");
    if(data.bin_fill_percent>80) suggestions.push("🗑 Bin almost full! Schedule collection.");
    document.getElementById('suggestions').innerHTML = suggestions.join('<br>');

    // Leaderboard
    const leaderboard = {Alice:10,Bob:15,Charlie:5};
    let html = '';
    Object.entries(leaderboard).sort((a,b)=>b[1]-a[1]).forEach(([user,pts])=>{
      html += <li>${user}: ${pts} pts</li>;
    });
    document.getElementById('leaderboard').innerHTML = html;

  } catch(e) {
    console.error("Error fetching data:", e);
  }
}

initCharts();
setInterval(fetchData,2000);
</script>
</body>
</html>
    """
    return render_template_string(html)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
