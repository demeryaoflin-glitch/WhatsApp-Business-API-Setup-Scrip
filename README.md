// file: App.jsx
// file: App.jsx
import React, {useState, useEffect, useRef} from "react";
import io from "socket.io-client";
import axios from "axios";

// الأفضل جعل عنوان السيرفر متغير بيئة
const SOCKET_URL = process.env.REACT_APP_SOCKET_URL || "http://127.0.0.1:8000";
const socket = io(SOCKET_URL);

export default function App(){
  const [telemetry, setTelemetry] = useState(null);
  const [commands, setCommands] = useState([]);
  const [cmdType, setCmdType] = useState("ATTITUDE_ADJUST");
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(()=>{
    const handleTelemetry = (msg) => {
      setTelemetry(msg.data);
    };
    
    const handleCommandUpdate = (msg) => {
      fetchCommands();
    };

    socket.on("telemetry", handleTelemetry);
    socket.on("command_update", handleCommandUpdate);
    
    // جلب البيانات الأولية
    fetchInitialData();

    return ()=> {
      socket.off("telemetry", handleTelemetry);
      socket.off("command_update", handleCommandUpdate);
    };
  }, []);

  const fetchInitialData = async () => {
    try {
      setLoading(true);
      const [commandsRes, telemetryRes] = await Promise.all([
        axios.get("/api/commands"),
        axios.get("/api/telemetry/latest")
      ]);
      setCommands(commandsRes.data);
      setTelemetry(telemetryRes.data.data);
    } catch (err) {
      setError("فشل في جلب البيانات");
      console.error("Error fetching data:", err);
    } finally {
      setLoading(false);
    }
  };

  const fetchCommands = async () => {
    try {
      const response = await axios.get("/api/commands");
      setCommands(response.data);
    } catch (err) {
      setError("فشل في تحديث الأوامر");
    }
  };

  const sendCommand = async () => {
    try {
      setLoading(true);
      const payload = { 
        type: cmdType, 
        params: { 
          note: "from web UI",
          timestamp: new Date().toISOString()
        } 
      };
      await axios.post("/api/commands", payload);
      // لا حاجة لـ console.log هنا في الإنتاج
    } catch (err) {
      setError("فشل في إرسال الأمر");
      console.error("Command error:", err);
    } finally {
      setLoading(false);
    }
  }

  if (loading && !telemetry) {
    return <div className="p-6">جاري التحميل...</div>;
  }

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">GCS — لوحة تحكم القمر (محاكاة)</h1>
      
      {error && (
        <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
          {error}
        </div>
      )}

      <div className="mb-6 p-4 border rounded">
        <h2 className="text-lg font-semibold mb-2">آخر تليمتري</h2>
        <pre className="bg-gray-100 p-3 rounded">
          {telemetry ? JSON.stringify(telemetry, null, 2) : "لا توجد بيانات"}
        </pre>
      </div>

      <div className="mb-6 p-4 border rounded">
        <h2 className="text-lg font-semibold mb-2">أرسل أمر</h2>
        <div className="flex items-center gap-2">
          <select 
            value={cmdType} 
            onChange={e=>setCmdType(e.target.value)}
            className="border p-2 rounded"
            disabled={loading}
          >
            <option value="ATTITUDE_ADJUST">ضبط الاتجاه</option>
            <option value="PAYLOAD_ON">تشغيل الحمولة</option>
            <option value="PAYLOAD_OFF">إيقاف الحمولة</option>
          </select>
          <button 
            onClick={sendCommand} 
            disabled={loading}
            className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600 disabled:bg-gray-400"
          >
            {loading ? "جاري الإرسال..." : "إرسال"}
          </button>
        </div>
      </div>

      <div className="p-4 border rounded">
        <h2 className="text-lg font-semibold mb-2">سجل الأوامر</h2>
        {commands.length === 0 ? (
          <p>لا توجد أوامر</p>
        ) : (
          <ul className="space-y-2">
            {commands.map(c=>(
              <li key={c.cmd_id} className="flex justify-between items-center p-2 bg-gray-50 rounded">
                <span className="font-mono">{c.cmd_id}</span>
                <span>{c.type}</span>
                <span className={`px-2 py-1 rounded ${
                  c.status === 'EXECUTED' ? 'bg-green-100 text-green-800' :
                  c.status === 'FAILED' ? 'bg-red-100 text-red-800' :
                  'bg-yellow-100 text-yellow-800'
                }`}>
                  {c.status}
                </span>
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  )
}import React, {useState, useEffect, useRef} from "react";
import io from "socket.io-client";
import axios from "axios";

const socket = io("http://127.0.0.1:8000");

export default function App(){
  const [telemetry, setTelemetry] = useState(null);
  const [commands, setCommands] = useState([]);
  const [cmdType, setCmdType] = useState("ATTITUDE_ADJUST");

  useEffect(()=>{
    socket.on("telemetry", (msg) => {
      setTelemetry(msg.data);
    });
    socket.on("command_update", (msg) => {
      // تحديث قائمة الأوامر: نجلب القائمة من السيرفر
      axios.get("/api/commands").then(r=> setCommands(r.data)).catch(()=>{});
    });
    // initial fetch
    axios.get("/api/commands").then(r=> setCommands(r.data)).catch(()=>{});
    axios.get("/api/telemetry/latest").then(r=> setTelemetry(r.data.data)).catch(()=>{});
    return ()=> socket.off("telemetry");
  }, []);

  const sendCommand = async () => {
    const payload = { type: cmdType, params: { note: "from web UI" } };
    const r = await axios.post("/api/commands", payload);
    console.log("cmd posted", r.data);
  }

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-4">GCS — لوحة تحكم القمر (محاكاة)</h1>
      <div className="mb-4">
        <h2 className="text-lg font-semibold">آخر تليمتري</h2>
        <pre>{JSON.stringify(telemetry, null, 2)}</pre>
      </div>

      <div className="mb-4">
        <h2 className="text-lg font-semibold">أرسل أمر</h2>
        <select value={cmdType} onChange={e=>setCmdType(e.target.value)}>
          <option>ATTITUDE_ADJUST</option>
          <option>PAYLOAD_ON</option>
          <option>PAYLOAD_OFF</option>
        </select>
        <button onClick={sendCommand} className="ml-2">إرسال</button>
      </div>

      <div>
        <h2 className="text-lg font-semibold">سجل الأوامر</h2>
        <ul>
          {commands.map(c=>(
            <li key={c.cmd_id}>{c.cmd_id} — {c.type} — {c.status}</li>
          ))}
        </ul>
      </div>
    </div>
  )
}pip install flask flask-sqlalchemy flask-socketio requestsconst express = require('express');gh repo clone demeryaoflin-glitch/WhatsApp-Business-API-Setup-Scrip
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(morgan('combined'));

let satelliteState = {
  latitude: 20.0,
  longitude: 30.0,
  status: "Nominal"
};

let commandHistory = []; // { command, result, timestamp }

function addHistory(command, result) {
  commandHistory.push({
    command,
    result,
    timestamp: new Date().toISOString()
  });
  // Keep history to a reasonable size
  if (commandHistory.length > 1000) commandHistory.shift();
}

app.get('/api/satellite/state', (req, res) => {
  res.json(satelliteState);
});

app.get('/api/satellite/history', (req, res) => {
  res.json(commandHistory);
});

app.post('/api/satellite/command', (req, res, next) => {
  try {
    const body = req.body;

    // Support both { command: "..." } and structured { type: "...", params: {...} }
    const commandRaw = body && (body.command || body);
    if (!commandRaw) {
      return res.status(400).json({ message: "يرجى تزويد الأمر (command)!" });
    }

    let resultMessage = "تم إرسال الأمر إلى القمر الصناعي بنجاح!";

    // If structured command object
    if (typeof commandRaw === 'object' && commandRaw.type) {
      const { type, params } = commandRaw;
      if (type === 'move' && params && typeof params.latitude === 'number' && typeof params.longitude === 'number') {
        satelliteState.latitude = params.latitude;
        satelliteState.longitude = params.longitude;
        satelliteState.status = `آخر أمر: move to (${params.latitude}, ${params.longitude})`;
        resultMessage = `قُصد التحرك إلى (${params.latitude}, ${params.longitude})`;
      } else {
        satelliteState.status = `آخر أمر: ${JSON.stringify(commandRaw)}`;
      }
      addHistory(commandRaw, resultMessage);
      return res.json({ message: resultMessage });
    }

    // If command is a string, try to parse simple "move <lat> <lon>" commands
    if (typeof commandRaw === 'string') {
      const command = commandRaw.trim();
      const moveMatch = command.match(/^move\s+(-?\d+(\.\d+)?)\s+(-?\d+(\.\d+)?)/i);
      if (moveMatch) {
        const lat = parseFloat(moveMatch[1]);
        const lon = parseFloat(moveMatch[3]);
        satelliteState.latitude = lat;
        satelliteState.longitude = lon;
        satelliteState.status = `آخر أمر: move to (${lat}, ${lon})`;
        resultMessage = `قُصد التحرك إلى (${lat}, ${lon})`;
      } else {
        satelliteState.status = `آخر أمر: ${command}`;
      }
      addHistory(command, resultMessage);
      return res.json({ message: resultMessage });
    }

    // Fallback (unknown shape)
    satelliteState.status = `آخر أمر: ${JSON.stringify(commandRaw)}`;
    addHistory(commandRaw, resultMessage);
    return res.json({ message: resultMessage });
  } catch (err) {
    next(err);
  }
});

// Basic error handler
app.use((err, req, res, next) => {
  console.error(err && err.stack ? err.stack : err);
  res.status(500).json({ message: "حدث خطأ في الخادم." });
});

// Graceful shutdown support
const PORT = process.env.PORT || 4000;
const server = app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));

function shutdown() {
  console.log('Shutting down server...');
  server.close(() => {
    console.log('Server closed.');
    process.exit(0);
  });
  // Force exit after 10s
  setTimeout(() => process.exit(1), 10000);
}
process.on('SIGINT', shutdown);
process.on('SIGTERM', shutdown);
