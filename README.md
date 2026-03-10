# ai-todo-app

Ai todo · JSX
Copy

import { useState, useRef, useEffect } from "react";

const ANTHROPIC_MODEL = "claude-sonnet-4-20250514";

const callClaude = async (systemPrompt, userMessage) => {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: ANTHROPIC_MODEL,
      max_tokens: 1000,
      system: systemPrompt,
      messages: [{ role: "user", content: userMessage }],
    }),
  });
  const data = await response.json();
  return data.content?.map(b => b.text || "").join("") || "";
};

const parseJSON = (text) => {
  try {
    const clean = text.replace(/```json|```/g, "").trim();
    return JSON.parse(clean);
  } catch {
    return null;
  }
};

let idCounter = 1;
const uid = () => `t${idCounter++}`;

export default function App() {
  const [tasks, setTasks] = useState([
    { id: uid(), text: "Try asking AI to add a task for you!", done: false, priority: "medium", tag: "demo" },
  ]);
  const [input, setInput] = useState("");
  const [chat, setChat] = useState([{ role: "ai", text: "Hi! I'm your AI assistant. Tell me what you need to do — I'll add tasks, prioritize them, or help you figure out what to tackle first. 🧠" }]);
  const [loading, setLoading] = useState(false);
  const [filter, setFilter] = useState("all");
  const chatEndRef = useRef(null);

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [chat]);

  const sendMessage = async () => {
    if (!input.trim() || loading) return;
    const userText = input.trim();
    setInput("");
    setChat(c => [...c, { role: "user", text: userText }]);
    setLoading(true);

    const taskList = tasks.map(t => `- [${t.done ? "x" : " "}] (${t.priority}) ${t.text}`).join("\n") || "No tasks yet.";

    const system = `You are a smart to-do list AI assistant. The user has these tasks:
${taskList}

You help the user manage their tasks. Based on the user's message, respond with a JSON object:
{
  "reply": "Your friendly reply to the user",
  "add": ["task text", ...],           // tasks to add (optional)
  "complete": ["partial task text"], // tasks to mark complete (optional)
  "delete": ["partial task text"],   // tasks to delete (optional)
  "prioritize": true                 // if user asks to prioritize, set true
}

When adding tasks, make them concise. For priority (low/medium/high), infer from context.
If user says something like "what should I do first" or "prioritize", set prioritize: true.
Only include fields that are needed. Always respond ONLY with valid JSON, no extra text.`;

    try {
      const raw = await callClaude(system, userText);
      const parsed = parseJSON(raw);

      if (!parsed) {
        setChat(c => [...c, { role: "ai", text: raw || "Sorry, I didn't understand that. Try again!" }]);
        setLoading(false);
        return;
      }

      let newTasks = [...tasks];

      if (parsed.add?.length) {
        const added = parsed.add.map(text => ({
          id: uid(), text, done: false, priority: "medium", tag: "task"
        }));
        newTasks = [...newTasks, ...added];
      }

      if (parsed.complete?.length) {
        newTasks = newTasks.map(t =>
          parsed.complete.some(c => t.text.toLowerCase().includes(c.toLowerCase()))
            ? { ...t, done: true } : t
        );
      }

      if (parsed.delete?.length) {
        newTasks = newTasks.filter(t =>
          !parsed.delete.some(d => t.text.toLowerCase().includes(d.toLowerCase()))
        );
      }

      if (parsed.prioritize) {
        // Ask AI to re-prioritize
        const priSystem = `Given these tasks, assign a priority (high/medium/low) to each one based on urgency and importance. Return ONLY a JSON array: [{"text": "...", "priority": "high|medium|low"}, ...]. Match text exactly.`;
        const priRaw = await callClaude(priSystem, newTasks.filter(t => !t.done).map(t => t.text).join("\n"));
        const priorities = parseJSON(priRaw);
        if (priorities) {
          newTasks = newTasks.map(t => {
            const match = priorities.find(p => p.text === t.text);
            return match ? { ...t, priority: match.priority } : t;
          });
          // Sort: high > medium > low
          const order = { high: 0, medium: 1, low: 2 };
          newTasks = [...newTasks].sort((a, b) => (order[a.priority] ?? 1) - (order[b.priority] ?? 1));
        }
      }

      setTasks(newTasks);
      setChat(c => [...c, { role: "ai", text: parsed.reply || "Done!" }]);
    } catch (e) {
      setChat(c => [...c, { role: "ai", text: "Something went wrong. Please try again." }]);
    }
    setLoading(false);
  };

  const toggleDone = (id) => setTasks(t => t.map(tk => tk.id === id ? { ...tk, done: !tk.done } : tk));
  const deleteTask = (id) => setTasks(t => t.filter(tk => tk.id !== id));
  const cyclePriority = (id) => {
    const cycle = { low: "medium", medium: "high", high: "low" };
    setTasks(t => t.map(tk => tk.id === id ? { ...tk, priority: cycle[tk.priority] } : tk));
  };

  const filtered = tasks.filter(t =>
    filter === "all" ? true : filter === "active" ? !t.done : t.done
  );

  const priorityColor = { high: "#ef4444", medium: "#f59e0b", low: "#10b981" };
  const priorityEmoji = { high: "🔴", medium: "🟡", low: "🟢" };

  return (
    <div style={{
      minHeight: "100vh",
      background: "linear-gradient(135deg, #0f0f1a 0%, #1a1025 50%, #0d1a2e 100%)",
      fontFamily: "'Georgia', serif",
      color: "#e2e8f0",
      display: "flex",
      flexDirection: "column",
      alignItems: "center",
      padding: "2rem 1rem",
    }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;700&family=DM+Sans:wght@300;400;500&display=swap');
        * { box-sizing: border-box; }
        body { margin: 0; }
        .task-row:hover { background: rgba(255,255,255,0.05) !important; }
        .task-row { transition: background 0.2s; }
        input:focus { outline: none; }
        ::-webkit-scrollbar { width: 4px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.2); border-radius: 2px; }
        .chip { cursor: pointer; transition: all 0.15s; }
        .chip:hover { opacity: 0.8; transform: translateY(-1px); }
        .send-btn:hover { background: rgba(139,92,246,0.4) !important; }
        .delete-btn:hover { color: #ef4444 !important; }
        .filter-btn { cursor: pointer; transition: all 0.15s; }
      `}</style>

      {/* Header */}
      <div style={{ textAlign: "center", marginBottom: "2rem" }}>
        <h1 style={{ fontFamily: "'Playfair Display', serif", fontSize: "2.5rem", margin: 0, background: "linear-gradient(90deg, #a78bfa, #60a5fa)", WebkitBackgroundClip: "text", WebkitTextFillColor: "transparent" }}>
          ✦ Task Mind
        </h1>
        <p style={{ fontFamily: "'DM Sans', sans-serif", fontWeight: 300, color: "#94a3b8", margin: "0.5rem 0 0", letterSpacing: "0.1em", fontSize: "0.85rem" }}>
          YOUR AI-POWERED TO-DO LIST
        </p>
      </div>

      <div style={{ display: "flex", gap: "1.5rem", width: "100%", maxWidth: "1000px", alignItems: "flex-start", flexWrap: "wrap" }}>

        {/* Tasks Panel */}
        <div style={{ flex: "1 1 380px", background: "rgba(255,255,255,0.04)", borderRadius: "1.25rem", border: "1px solid rgba(255,255,255,0.08)", padding: "1.5rem", backdropFilter: "blur(12px)" }}>
          <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "1.25rem" }}>
            <h2 style={{ fontFamily: "'Playfair Display', serif", margin: 0, fontSize: "1.2rem" }}>Tasks</h2>
            <div style={{ display: "flex", gap: "0.4rem" }}>
              {["all", "active", "done"].map(f => (
                <button key={f} className="filter-btn" onClick={() => setFilter(f)} style={{
                  background: filter === f ? "rgba(139,92,246,0.3)" : "transparent",
                  border: `1px solid ${filter === f ? "#a78bfa" : "rgba(255,255,255,0.1)"}`,
                  color: filter === f ? "#a78bfa" : "#94a3b8",
                  borderRadius: "999px", padding: "0.2rem 0.7rem", fontSize: "0.75rem",
                  fontFamily: "'DM Sans', sans-serif", cursor: "pointer",
                }}>
                  {f.charAt(0).toUpperCase() + f.slice(1)}
                </button>
              ))}
            </div>
          </div>

          <div style={{ display: "flex", flexDirection: "column", gap: "0.5rem", minHeight: "100px" }}>
            {filtered.length === 0 && (
              <div style={{ color: "#475569", textAlign: "center", padding: "2rem 0", fontFamily: "'DM Sans', sans-serif", fontSize: "0.9rem" }}>
                No tasks here yet. Ask the AI to add some! ✨
              </div>
            )}
            {filtered.map(task => (
              <div key={task.id} className="task-row" style={{
                display: "flex", alignItems: "center", gap: "0.75rem",
                padding: "0.6rem 0.75rem", borderRadius: "0.75rem",
                background: "rgba(255,255,255,0.02)", border: "1px solid rgba(255,255,255,0.05)",
              }}>
                <input type="checkbox" checked={task.done} onChange={() => toggleDone(task.id)}
                  style={{ accentColor: "#a78bfa", width: "1rem", height: "1rem", cursor: "pointer", flexShrink: 0 }} />
                <span style={{
                  flex: 1, fontFamily: "'DM Sans', sans-serif", fontSize: "0.9rem",
                  textDecoration: task.done ? "line-through" : "none",
                  color: task.done ? "#475569" : "#e2e8f0",
                }}>
                  {task.text}
                </span>
                <button className="chip" onClick={() => cyclePriority(task.id)} title="Click to change priority" style={{
                  background: "transparent", border: "none", cursor: "pointer", padding: "0 0.1rem", fontSize: "0.8rem"
                }}>
                  {priorityEmoji[task.priority]}
                </button>
                <button className="delete-btn" onClick={() => deleteTask(task.id)} style={{
                  background: "transparent", border: "none", color: "#475569",
                  cursor: "pointer", fontSize: "1rem", padding: 0, lineHeight: 1,
                }}>×</button>
              </div>
            ))}
          </div>

          <div style={{ marginTop: "1rem", paddingTop: "1rem", borderTop: "1px solid rgba(255,255,255,0.06)", display: "flex", gap: "1rem", fontFamily: "'DM Sans', sans-serif", fontSize: "0.78rem", color: "#475569" }}>
            <span>{tasks.filter(t => !t.done).length} remaining</span>
            <span>{tasks.filter(t => t.done).length} completed</span>
            <span style={{ marginLeft: "auto" }}>🔴 High · 🟡 Med · 🟢 Low</span>
          </div>
        </div>

        {/* AI Chat Panel */}
        <div style={{ flex: "1 1 340px", background: "rgba(255,255,255,0.04)", borderRadius: "1.25rem", border: "1px solid rgba(255,255,255,0.08)", padding: "1.5rem", backdropFilter: "blur(12px)", display: "flex", flexDirection: "column", gap: "1rem" }}>
          <h2 style={{ fontFamily: "'Playfair Display', serif", margin: 0, fontSize: "1.2rem" }}>✦ AI Assistant</h2>

          {/* Chat */}
          <div style={{ height: "300px", overflowY: "auto", display: "flex", flexDirection: "column", gap: "0.6rem" }}>
            {chat.map((msg, i) => (
              <div key={i} style={{
                alignSelf: msg.role === "user" ? "flex-end" : "flex-start",
                background: msg.role === "user" ? "rgba(139,92,246,0.25)" : "rgba(255,255,255,0.06)",
                border: `1px solid ${msg.role === "user" ? "rgba(139,92,246,0.4)" : "rgba(255,255,255,0.08)"}`,
                borderRadius: "1rem", padding: "0.6rem 0.9rem",
                maxWidth: "85%", fontFamily: "'DM Sans', sans-serif",
                fontSize: "0.87rem", lineHeight: "1.5", color: "#e2e8f0",
              }}>
                {msg.text}
              </div>
            ))}
            {loading && (
              <div style={{ alignSelf: "flex-start", background: "rgba(255,255,255,0.06)", border: "1px solid rgba(255,255,255,0.08)", borderRadius: "1rem", padding: "0.6rem 0.9rem", fontFamily: "'DM Sans', sans-serif", fontSize: "0.87rem", color: "#475569" }}>
                Thinking...
              </div>
            )}
            <div ref={chatEndRef} />
          </div>

          {/* Suggestions */}
          <div style={{ display: "flex", gap: "0.4rem", flexWrap: "wrap" }}>
            {["Add a task for me", "What should I do first?", "Prioritize my list", "Mark all done"].map(s => (
              <button key={s} className="chip" onClick={() => setInput(s)} style={{
                background: "rgba(255,255,255,0.05)", border: "1px solid rgba(255,255,255,0.1)",
                color: "#94a3b8", borderRadius: "999px", padding: "0.25rem 0.7rem",
                fontSize: "0.72rem", fontFamily: "'DM Sans', sans-serif", cursor: "pointer",
              }}>
                {s}
              </button>
            ))}
          </div>

          {/* Input */}
          <div style={{ display: "flex", gap: "0.5rem" }}>
            <input
              value={input}
              onChange={e => setInput(e.target.value)}
              onKeyDown={e => e.key === "Enter" && sendMessage()}
              placeholder="Ask AI to manage your tasks..."
              style={{
                flex: 1, background: "rgba(255,255,255,0.06)", border: "1px solid rgba(255,255,255,0.1)",
                borderRadius: "0.75rem", padding: "0.6rem 1rem", color: "#e2e8f0",
                fontFamily: "'DM Sans', sans-serif", fontSize: "0.87rem",
              }}
            />
            <button className="send-btn" onClick={sendMessage} disabled={loading} style={{
              background: "rgba(139,92,246,0.25)", border: "1px solid rgba(139,92,246,0.4)",
              borderRadius: "0.75rem", padding: "0 1rem", color: "#a78bfa",
              cursor: loading ? "not-allowed" : "pointer", fontSize: "1.1rem",
            }}>
              ↑
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
